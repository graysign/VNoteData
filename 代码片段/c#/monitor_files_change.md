```c#
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;

namespace FileMonitor
{
    internal class Program
    {
        private static void Main(string[] args)
        {
            WatcherStrat(@"D:\", "*.*");
            Console.ReadKey();
        }

        private static void WatcherStrat(string path, string filter)
        {
            FileSystemWatcher watcher = new FileSystemWatcher();
            watcher.Path = path;
            watcher.Filter = filter;
            watcher.Changed += new FileSystemEventHandler(OnProcess);
            watcher.Created += new FileSystemEventHandler(OnProcess);
            watcher.Deleted += new FileSystemEventHandler(OnProcess);
            watcher.Renamed += new RenamedEventHandler(OnRenamed);
            watcher.EnableRaisingEvents = true;
            watcher.NotifyFilter = NotifyFilters.Attributes | NotifyFilters.CreationTime | NotifyFilters.DirectoryName | NotifyFilters.FileName | NotifyFilters.LastAccess
                                   | NotifyFilters.LastWrite | NotifyFilters.Security | NotifyFilters.Size;
            watcher.IncludeSubdirectories = true;
        }

        private static void OnProcess(object source, FileSystemEventArgs e)
        {
            if (e.ChangeType == WatcherChangeTypes.Created)
            {
                OnCreated(source, e);
            }
            else if (e.ChangeType == WatcherChangeTypes.Changed)
            {
                OnChanged(source, e);
            }
            else if (e.ChangeType == WatcherChangeTypes.Deleted)
            {
                OnDeleted(source, e);
            }
        }

        private static void OnCreated(object source, FileSystemEventArgs e)
        {
            Console.WriteLine("{0}  {1}  {2}", e.ChangeType, e.FullPath, e.GetHashCode());
        }

        private static void OnChanged(object source, FileSystemEventArgs e)
        {
            Console.WriteLine("{0}  {1}  {2}", e.ChangeType, e.FullPath, e.GetHashCode());
        }

        private static void OnDeleted(object source, FileSystemEventArgs e)
        {
            Console.WriteLine("{0}  {1}   {2}", e.ChangeType, e.FullPath, e.GetHashCode());
        }

        private static void OnRenamed(object source, RenamedEventArgs e)
        {
            Console.WriteLine("{0}  {1}  {2}", e.ChangeType, e.FullPath, e.GetHashCode());
        }
    }
}
```