* 首先, 设置list 列表的风格
    `void ListView_SetExtendedListViewStyle(m_lvTestList.m_hWnd, LVS_EX_CHECKBOXES | LVS_EX_FULLROWSELECT);`  
* 使得list 控件支持`checkbox LVS_EX_CHECKBOXESlist` 的每一个item 都可以使用checkbox 控件, 可以通过使用宏`ListView_GetCheckState` 来获得checkbox 的状态  
    * LVS_EX_FULLROWSELECT当一个item 被选中时, 它的所有subitems 也处于被选中状态, 点击任意一个subitem, 则可同时选中整个行. 只适用于`LVS_REPORT` 风格
    * LVS_EX_GRIDLINES网格线, 只适用于LVS_REPORT 风格
    * LVS_EX_HEADERDRAGDROP支持列头的拖拽, 只适用于LVS_REPORT 风格
    * LVS_EX_SUBITEMIMAGES可在subitem 中插入图标 , 只适用于LVS_REPORT 风格
    * LVS_EX_TRACKSELECT如果鼠标停留在某个item 上超过1 秒钟, 则此item 显示为被选中状态. 适用于任何风格的List 控件
* 当一个checkbox 被check 或uncheck 的时候, 如何获得通知 添加消息映射 `ON_NOTIFY(LVN_ITEMCHANGED, IDC_MYLIST, OnItemchangedLinksList)`
消息处理函数
```MFC
void DemoDlg::OnItemchangedLinksList(NMHDR* pNMHDR, LRESULT* pResult) 
{
    NM_LISTVIEW* pNMListView = (NM_LISTVIEW*)pNMHDR;
    *pResult = 0;

    if (pNMListView->uOldState == 0 && pNMListView->uNewState == 0)
        return;    // No change

    // Old check box state
    BOOL bPrevState = (BOOL)(((pNMListView->uOldState & 
                LVIS_STATEIMAGEMASK)>>12)-1);  
    if (bPrevState < 0)    // On startup there's no previous state 
        bPrevState = 0; // so assign as false (unchecked)

    // New check box state
    BOOL bChecked = 
         (BOOL)(((pNMListView->uNewState &LVIS_STATEIMAGEMASK)>>12)-1);   
    if (bChecked < 0) // On non-checkbox notifications assume false
        bChecked = 0;

    if (bPrevState == bChecked) // No change in check box
        return;    // Now bChecked holds the new check box state

    // ....
}
```
* 设置某个item 的checkbox 的状态
```MFC
void SetLVCheck (WPARAM ItemIndex, BOOL bCheck)
{
    ListView_SetItemState (m_lvTestList.m_hWnd, ItemIndex, UINT((int(bCheck) + 1) << 12), LVIS_STATEIMAGEMASK);
}
```
* 获得某个item 的checkbox 的状态 使用宏`ListView_GetCheckState(hwndLV, i)`
目的：使列表框（CListCtrl）的每个项(item)前面有个复选，用来标明是否选中，提交数据是只选择选中的  
方法：为列表框（CListCtrl）多加一个`LVS_EX_CHECKBOXES`风格  

```MFC
m_list.SetExtendedStyle(LVS_EX_GRIDLINES|LVS_EX_FULLROWSELECT|LVS_EX_CHECKBOXES);
 //添加的项（即“行”）的第一列总会在项目名前出现一个复选框（添加项后才能看到复选框）
 m_list.InsertColumn(0,"选取",LVCFMT_LEFT,50);        //添加列标题
 m_list.InsertColumn(1,"1",LVCFMT_LEFT,50);
 m_list.InsertColumn(2,"2",LVCFMT_LEFT,50);
 m_list.InsertColumn(3,"3",LVCFMT_LEFT,50);

 m_list.InsertItem(0,"");   //添加项（即行标题）
 m_list.InsertItem(1,"");
 
 m_list.SetItemText(0,1,"sdf");
 m_list.SetItemText(1,0,"sdf");  //设置项的各列数据时也可更改行标题，即行的第一列的文字
 m_list.SetItemText(1,1,"sdf");
 m_list.SetItemText(1,2,"sdf");
 m_list.SetItemText(1,3,"sdf");

/**********************************************************************************
也可以使用以下方法设置数据*/

   m_list.SetBkColor(RGB(255,255,255));     //设置背景
   m_list.SetTextBkColor(RGB(255,255,255));  
 
   //设置list对话框的列   
   LV_COLUMN   lvc;  

   lvc.mask=LVCF_TEXT|LVCF_SUBITEM|LVCF_WIDTH|LVCF_FMT;  
   lvc.fmt=LVCFMT_LEFT;  
   
   lvc.iSubItem=0;  
  lvc.pszText=_T("A");  
   lvc.cx=50;  
   m_list.InsertColumn(1,&lvc);  
   
   lvc.iSubItem=1;  
   lvc.pszText=_T("B");  
   lvc.cx=50;  
   m_list.InsertColumn(2,&lvc);
```