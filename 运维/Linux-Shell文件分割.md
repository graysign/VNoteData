```sh
#!/bin/bash

export PATH="/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"
export strProgName=`basename $0`
export nProgNameStat=1
function Usage()
{
cat << END
******************************************************************************
* Describe: This script is used to split large log file
* Author:   zhangyuanwu
* Time:     2020/01/09
* Uage: 
*       $strProgName <File>  [SplitSize(M) default 10]  [SaveLogCount default 20]
* E.g.:
*       $strProgName xxxxxxxx.log  
*       $strProgName xxxxxxxx.log  10  20
******************************************************************************
END
}


function DealSignals() {
    Log "Recive Signal Exit Wait....................!"
    nProgNameStat=0
}

function Log()
{
    local strTime=`date "+%Y-%m-%d %H:%M:%S"`
    echo -e "$strTime    Info    $*"
}

function GetDateBy8Zone()
{
    local strTimeZone=`date -R|awk '{print $6}'`
    local strDateStamp=`date "+%Y-%m-%d"`
    if [ $strTimeZone -eq "+0000" ]; then
        let nTimeStamp=$((`date +%s`+8*3600))
        strDateStamp=`date -d @$nTimeStamp '+%Y-%m-%d'`
    fi
    echo "$strDateStamp"
}

function CheckFileIsMonitor()
{
    local strProgName=$1
    local strFile=$2
    let   nScriptPid=$3
    local nScriptPidArr=(`ps -ef|grep "${strProgName}"|grep "${strFile}"|grep -Ev grep|awk '{print $2}'`)

    for nPid in ${nScriptPidArr[@]};
    do
        if [ $nPid -eq $nScriptPid ]; then
            continue
        fi
        local nCount=`ps -ef|grep -Ev grep|awk '{print $2}'|grep -w ${nPid}|wc -l`
        if [ $nCount -eq 0 ]; then
            continue
        fi
        Log "Find this file is monitor,then exit!"
        exit
    done
    
}

function CheckAndPackFile()
{
    local strFileName=$1
    local strDateStamp=$2
    local strPackLogFilesDateArr=(`ls -lt |awk '{print $9}'|grep -E "^Log_....._....-..-.._${strFileName}"|awk -F_ '{print $3}'`)
    declare -A strDateFileMap
    for strPackDateStamp in ${strPackLogFilesDateArr[@]};
    do
        if [ $strPackDateStamp == $strDateStamp ]; then
            continue
        fi
        strDateFileMap[$strPackDateStamp]=1
    done
    for strPackDateStampKey in ${!strDateFileMap[*]};do
        local strPackFileName="${strPackDateStampKey}_${strFileName}.tgz"
        local tarCommand=`tar -zcf "$strPackFileName" Log_*****_${strPackDateStampKey}_${strFileName} --remove-files` #>/dev/null 2>&1
        #echo $tarCommand
    done

}

function SplitFile()
{
    local strFileFullPath=$1
    local strFileName=$2
    local nSplitSize=$3
    local strDateStamp=$4
    #chek file size
    let nCurrentSize=`ls -l "$strFileFullPath"|awk '{print $5}'`
    if [ $nCurrentSize -lt $nSplitSize ]; then
        #echo "FileSize $CurrentSize < $SplitSize"
        return 1
    fi
    
    #calc split file count
    local strProcessingFile="$strFileName.tmp"
    local strRemainingBitFile="$strFileName.last"
    if [ -f "$strProcessingFile" ]; then
        rm -rf "$strProcessingFile"
    fi 
    mv -f "$strFileName" "$strProcessingFile"
    let nCurrentSize=`ls -l "$strProcessingFile"|awk '{print $5}'`
    let nSplitCount=$(($nCurrentSize/$nSplitSize))
    let nRemainingBit=$(($nCurrentSize%$nSplitSize))

    #split file
    if [ $nRemainingBit -gt 0 ]; then
        dd if="$strProcessingFile" bs=$nSplitSize count=1 skip=$nSplitCount of="$strRemainingBitFile" >/dev/null 2>&1
        if [ -f "$strFileName" ]; then
            cat "$strFileName" >> "$strRemainingBitFile"
        fi
        mv -f "$strRemainingBitFile" "$strFileName"
    else
        touch "$strFileName"
    fi

    let nSplitTimeLast=`date +%s`
    let nSplitTimePre=$(($SplitTimeLast-1))
    local strCurrentIndex=`ls -lt |awk '{print $9}'|grep -E "^Log_....._${strDateStamp}_${strFileName}"|awk -F_ '{print $2}'|head -1 `
    if [ -n "$strCurrentIndex" ];then
       for((i=0;i<${#strCurrentIndex};i++));do
            if [ ${strCurrentIndex:$i:1} == "0" ];then
                continue
            fi
            strCurrentIndex=${strCurrentIndex:$i}
            break;
       done
    else
        strCurrentIndex=99999
    fi

    for((i=0;i<$nSplitCount;i++));  
    do   
        if [ $strCurrentIndex -eq 99999 ] ; then
            strCurrentIndex=0
        else
            strCurrentIndex=$((10#$strCurrentIndex+1))
        fi
        
        local strFileFullName="Log_`printf %05d ${strCurrentIndex}`_${strDateStamp}_${strFileName}"
        dd if="$strProcessingFile" bs=$nSplitSize count=1 skip=$i of="$strFileFullName"    >/dev/null 2>&1

        #mcu set file modif time
        if [ -f "/bin/busybox" ]; then
            local strFileModifTime=`date "+%Y-%m-%d %H:%M:%S" -d @${nSplitTimePre}`
            if [ $i -eq $(($nSplitCount-1)) ]; then
                  strFileModifTime=`date "+%Y-%m-%d %H:%M:%S" -d @${nSplitTimeLast}`
            fi
            touch -m -d "$strFileModifTime" "$strFileFullName"
        fi
    done  

    #delete tmp file
    rm -rf "$strProcessingFile"
    return 0
}

function ClearFile()
{
    local strFilePath=$1
    local strFileName=$2
    local strDateStamp=$3
    local nSaveFileCount=$4
    local strCurrentDayLogFilesArr=(`ls -lt |awk '{print $9}'|grep -E "^Log_....._${strDateStamp}_${strFileName}"`)

    if [ ${#strCurrentDayLogFilesArr[*]} -lt $nSaveFileCount ];then
        return 1
    fi
    for((i=0;i<${#strCurrentDayLogFilesArr[*]};i++));
    do
        if [ $i -lt $nSaveFileCount ];then
            continue
        fi
        rm  -rf    "${strFilePath}/${strCurrentDayLogFilesArr[$i]}"
    done
}

function main()
{
    if [ $# -lt 1 ];then
        Usage
        exit
    fi

    local strFileFullPath=$1
    local strFilePath=`dirname $strFileFullPath`
    local strFileName=`basename $strFileFullPath`
    local nSplitSize=$((10*1024*1024))
    local nWaitSecond=20
    if [ $# -eq 2 ] && [ $2 -gt 0]  2>/dev/null;then
          nSplitSize=$(($2*1024*1024))
    fi
    local nSaveFileCount=$((20+0))
    if [ $# -eq 3 ] && [ $3 -gt 0]  2>/dev/null;then
          nSaveFileCount=$(($3+0))
    fi

    #check this file  script is monitor
    CheckFileIsMonitor "$strProgName" "$strFileFullPath" $$
    

    cd "$strFilePath"

    Log "FilePath: $strFilePath "
    Log "FileName: $strFileName "
    Log "SplitSize: $nSplitSize "
    Log "SaveFileCount: $nSaveFileCount "

    while true
    do
        Log "Wait $nWaitSecond second.."
        sleep $nWaitSecond
        #check file
        if [ ! -f "$strFileFullPath" ]; then
            Log "$strFileFullPath Not Exist!"
            continue
        fi
        local strDateStamp=$(GetDateBy8Zone)
        #check and pack file
        Log "CheckAndPackFile   Start ..."
        CheckAndPackFile "$strFileName" "$strDateStamp"
        Log "CheckAndPackFile   Finsh ..."


        Log "ClearFile  Start ..."
        #delete greater than $SaveFileCount file
        ClearFile "$strFilePath" "$strFileName" "$strDateStamp" "$nSaveFileCount"
        Log "ClearFile  Finsh ..."


        Log "SplitFile  Start ..."
        #check and split file
        SplitFile "$strFileFullPath" "$strFileName" $nSplitSize "$strDateStamp"
        Log "SplitFile  Finsh ..."

        if [ $nProgNameStat -eq 0 ];then
            exit
        fi
    done
}
#signal 2  ctrl+c
#signal 15 SIGTERM
trap "DealSignals" 2 15 
main $*
```