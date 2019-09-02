```c#
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Windows.Forms;

namespace TreeViewDemo
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
            TreeViewConfig();

            TreeNode Root = treeView1.Nodes.Add("根");
            TreeNode Root2 = treeView1.Nodes.Add("根2");

            TreeNode Root1 = Root.Nodes.Add("根11");
            AddDeleteButtonChildNode(Root1, "测试1");
            AddDeleteButtonChildNode(Root1, "测试2");
            AddDeleteButtonChildNode(Root1, "测试3");

            AddDeleteButtonChildNode(Root2, "测试111111111");
            AddDeleteButtonChildNode(Root2, "测试2111111111111");
            AddDeleteButtonChildNode(Root2, "测试3111111111111");
            
        }


        //treeview 设置
        private void TreeViewConfig()
        {
            
            treeView1.ItemHeight = 20;//node节点的高度
            treeView1.Dock = DockStyle.Fill;
            treeView1.AfterExpand += new TreeViewEventHandler(TV_AfterExpand);//处理展开事件
            treeView1.AfterCollapse += new TreeViewEventHandler(TV_AfterCollapse);//处理折叠事件
            treeView1.AfterLabelEdit += new NodeLabelEditEventHandler(TV_AfterLabelEdit);//节点名字修改事件
             
            treeView1.LabelEdit = true;
        }
        
        void TV_AfterCollapse(object sender, TreeViewEventArgs e)
        {
            
            //ChangeDeleteButtonVisible(e.Node, false);
            ChangeDeleteButtonVisible(); 
        }

        void TV_AfterExpand(object sender, TreeViewEventArgs e)
        {
            
            //ChangeDeleteButtonVisible(e.Node, true);
             ChangeDeleteButtonVisible();
       
        }
        void TV_AfterLabelEdit(object sender, NodeLabelEditEventArgs e)
        {
            ChangeNodeStatusByLableEdit(e.Node,e.Label);
        }

        void ChangeNodeStatusByLableEdit(TreeNode ChildNode,string newtext)
        {
            if (ChildNode.Tag is TreeItem)
            {
                Graphics g = treeView1.CreateGraphics();
                SizeF StrSize = g.MeasureString(newtext, treeView1.Font);
                TreeItem btns = (TreeItem)ChildNode.Tag;
                int width = (int)StrSize.Width;
                foreach (PictureBox b in btns.btns)
                {

                    b.Location = ChildNode.Bounds.Location;
                    b.Left = b.Left  + width;
                    b.Visible = true;
                    width = b.Width + width + btns.space;
                }
            }
        }
        void ChangeNodeStatus(TreeNode ParentNode, bool isVisible)
        {
            foreach (TreeNode ChildNode in ParentNode.Nodes)
            { 
                if (ChildNode.Tag is TreeItem)
                {
                    TreeItem btns = (TreeItem)ChildNode.Tag;
                    int width = ChildNode.Bounds.Width;

                    foreach (PictureBox b in btns.btns)
                    {

                        b.Location = ChildNode.Bounds.Location;
                        b.Left = b.Left + width;
                        if (isVisible == false)
                        {
                            b.Visible = false;
                        }
                        else
                        {
                            b.Visible = ParentNode.IsExpanded;
                        }
                        
                        width = b.Width + width + btns.space;
                    }
                }
                ChangeNodeStatus(ChildNode,isVisible);
                /*Button B = (Button)ChildNode.Tag;
                B.Location = ChildNode.Bounds.Location;
                B.Left += ChildNode.Bounds.Width;
                B.Visible = Visible;*/
            }
        }
        //折叠 展开的时候 处理按钮 属性
        void ChangeDeleteButtonVisible()
        {
            foreach (TreeNode ChildNode in treeView1.Nodes)
            {
                ChangeNodeStatus(ChildNode, ChildNode.IsExpanded);
            }

            
        }

        //在子节点 增加  按钮
        void AddDeleteButtonChildNode(TreeNode ParentNode, String Text)
        {
            //添加text
            TreeNode Node = ParentNode.Nodes.Add(Text);
            
            //设置两个按钮
            TreeItem tag = new TreeItem();//自定义类，可以增加n个按钮，根据需求调整数量
            tag.space = 10;
            
            
            PictureBox p = new PictureBox();
            p.Load("D://43821.jpg");
            p.Width = 40;
            p.Height = 10;
            p.Visible = false;
            p.Parent = treeView1;
            tag.btns.Add(p);//


            PictureBox p2 = new PictureBox();
            p2.Load("D://43821.jpg");
            p2.Width = 40;
            p2.Height = 10;
            p2.Visible = false;
            p2.Parent = treeView1;
            tag.btns.Add(p2);//

            /*
            //按钮1
            Button B = new Button();
            B.Parent = treeView1;
            B.Text = "飞行";
            B.Visible = false;
            B.Click += new EventHandler(B_Click);//设置按钮单击事件
            B.Tag = "fly";//设置按钮上的附加数据，根据需求 可以把参数 设置上去
            B.Width = 40;//按钮宽度
            tag.btns.Add(B);//

            //按钮2
            Button A = new Button();
            A.Parent = treeView1;
            A.Text = "动画";
            A.Visible = false;
            A.Click += new EventHandler(B_Click);
            A.Tag = "ansi";
            A.Width = 40;
            tag.btns.Add(A);
            
            */

            Node.Tag = tag;

        }

        void B_Click(object sender, EventArgs e)
        {
            Button B = (Button)sender;
            MessageBox.Show((String)B.Tag);//根据需求 处理参数
            //TreeNode Node = (TreeNode)B.Tag;
            //TreeNode ParentNode = Node.Parent;
            //B.Dispose();
            //Node.Remove();
            //ChangeDeleteButtonVisible(ParentNode, true); // 刷新重新定位按钮
        }

        private class TreeItem
        {
            public List<PictureBox> btns = new List<PictureBox>();
            public int space = 0;//button之间的距离
        }
    }
}

```