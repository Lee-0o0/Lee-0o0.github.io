---
title: WinForm如何实现ComboBox模糊查询
date: 2023-02-16 10:40:23
tags: ["C#","IT技术","WinForm","转载文章"]
categories: ["IT技术"]
---

> 作者：[枫上善若水](http://www.cnblogs.com/xilipu31/)
> 出处：http://www.cnblogs.com/xilipu31/
> 本文版权归作者和博客园共有,欢迎转载,但未经作者同意必须保留此段声明,且在文章页面明显位置给出原文链接。

前台设计：

　　前台就是一个简单的Form窗体+一个ComboBox控件。

思路整理：

1. 用一个`List<string> listOnit`存放初始化数据，用一个`List<string> listNew`存放输入key之后，返回的数据。

2. 用上面的`listOnit`初始化ComboBox数据源进行绑定。

  3. 在TextUpdate方法内部，添加实现方法。

     首先进入方法，先清除ComboBox的内容，然后将输入的内容去`listOnit`初始化的数据中比对，找出对应数据，然后放入`listNew`存放数据，最后将`listNew`数据重新赋值给ComboBox。

源码：

```c#
namespace TimerDemo
{
    public partial class Form2 : Form
    {
        //初始化绑定默认关键词（此数据源可以从数据库取）
        List<string> listOnit = new List<string>();
        //输入key之后，返回的关键词
        List<string> listNew = new List<string>();
 
        public Form2()
        {
            InitializeComponent();
        }
 
        private void Form2_Load(object sender, EventArgs e)
        {
            //调用绑定
            BindComboBox();
        }
        /// <summary>
        /// 绑定ComboBox
        /// </summary>
        private void BindComboBox()
        {
            listOnit.Add("张三");
            listOnit.Add("张思");
            listOnit.Add("张五");
            listOnit.Add("王五");
            listOnit.Add("刘宇");
            listOnit.Add("马六");
            listOnit.Add("孙楠");
            listOnit.Add("那英");
            listOnit.Add("刘欢");
 
            /*
             * 1.注意用Item.Add(obj)或者Item.AddRange(obj)方式添加
             * 2.如果用DataSource绑定，后面再进行绑定是不行的，即便是Add或者Clear也不行
             */
            this.comboBox1.Items.AddRange(listOnit.ToArray());
        }
 
        private void comboBox1_TextChanged(object sender, EventArgs e)
        {
            /*
              
             * 不能用TextChanged操作，当this.comboBox1.DroppedDown为True时，选择项上下键有冲突
              
             */
 
        }
 
        private void comboBox1_TextUpdate(object sender, EventArgs e)
        {
            //清空combobox
            this.comboBox1.Items.Clear();
            //清空listNew
            listNew.Clear();
            //遍历全部备查数据
            foreach (var item in listOnit)
            {
                if (item.Contains(this.comboBox1.Text))
                {
                    //符合，插入ListNew
                    listNew.Add(item);
                }
            }
            //combobox添加已经查到的关键词
            this.comboBox1.Items.AddRange(listNew.ToArray());
            //设置光标位置，否则光标位置始终保持在第一列，造成输入关键词的倒序排列
            this.comboBox1.SelectionStart = this.comboBox1.Text.Length;
            //保持鼠标指针原来状态，有时候鼠标指针会被下拉框覆盖，所以要进行一次设置。
            Cursor = Cursors.Default;
            //自动弹出下拉框
            this.comboBox1.DroppedDown = true;
        }
    }
}
```

效果：

![img](/img/Winform%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0ComboBox%E6%A8%A1%E7%B3%8A%E6%9F%A5%E8%AF%A2/261053162014804.png)

实现过程中的问题：

　　1.绑定数据一开始用的DataSource方式，但是写到下面重新给ComboBox设置数据源的时候，报错：不能为已经设置DataSource的combobox赋值。

　　　　　　解决方式：将赋值方式改为：Item.Add(obj)或者Item.AddRange(obj)方式

　　2.下拉框的内容一直在增加

　　　　　　解决方式：当文本框文本改变时，清空下拉框的内容，然后再添加数据。

　　3.输入文本改变时，没有自动弹出下拉框显示已经查询好的数据。

　　　　　　解决方式：设置comboBox的DroppedDown 属性为True。

　　4.ComboBox文本框改变事件一开始选择用的是TextChanged事件，但是当在界面用 上 下键盘选择时，出现bug，不能进行选择。

　　　　　　解决方式：将文本框改变事件换为TextUpdate事件，然后添加实现方法。

　　5.当在ComboBox输入内容时，内容文本是倒序输出的，光标位置始终在最前面。

　　　　　　解决方式：设置光标的显示位置，this.comboBox1.SelectionStart = this.comboBox1.Text.Length;

　　6.输入内容改变时，用鼠标选择下拉列表项的时候，鼠标指针消失，被下拉框覆盖掉。

　　　　　　解决方式：设置鼠标状态为一开始的默认状态，Cursor = Cursors.Default;
