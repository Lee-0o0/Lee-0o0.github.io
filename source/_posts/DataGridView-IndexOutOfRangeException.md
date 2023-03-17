---
title: 'DataGridView:IndexOutOfRangeException 索引-1没有值'
date: 2023-03-17 10:53:45
tags: ["C#", "WinForm", "DataGridView", "IT技术", "转载文章"]
categories: ["IT技术"]
---

本文系转载文章，原文作者信息：

> 作者：[BadTree](https://home.cnblogs.com/u/badtree/)
>
> 出处：https://www.cnblogs.com/badtree/articles/1799170.html
>
> 本文版权归作者和博客园共有,欢迎转载,但未经作者同意必须保留此段声明,且在文章页面明显位置给出原文链接。

很多WinForm的开发人员在`DataGridView`的开发当中，都会出现<span style="color:red">“ 索引-1没有值 ”</span>这个烦人的问题，其实较早之前，我已经大概知道问题的所在，也找到了解决方法，不过一直没有时间去深入研究一下，今日做了一个测试，发现了问题的所在，我不知道这个问题是否应为MS的BUG，但至少我个人认为这个问题不应该出现！

下面先说说构成这个错误的现象。

首先出现这个错误，绝大多数的开发人员都是在进行数据绑定之后出现的，而且出现的情况基本上都只是一种，就是开始绑定的数据集是非空的，但数据集的 Count=0，在将这个非空的而元素个数为0的数据集绑定到`DataGridView`后，当更新`DataGridView`的数据源，即将一个元素个数大于0的数据集绑定给`DataGridView`后，`DataGridView`仍能正常显示，以上还是正常的，但问题就出在，当你用鼠标点击`DataGridView`后，<span style="color:red">“ 索引-1没有值 ”</span>这个恼人的错误就会出现。

其实以上的文字基本上已经让你知道问题的所在，就是第一次绑定的“非空的且元素个数为0的数据集”，经运行时查看对象属性，由于只要数据集不为空，`DataGirdView`就必需指定当前单元格（CurrentCell），但“非空但大小为零的数据集”的CurrentCell是为null，致使后来更新数据集后，这个CurrentCell仍不会变，因为你的数据集没有改变，只是数据集的数目改变了，所以CurrentCell不变，所以当你点击鼠标进去后，返回的当前行就出错了！

解决的方法很简单，

- 第一，绑定数据集时，判断数据集是否为空，是否元素个数大于0，如果符合条件的才将数据集绑定；

  ```c#
  if (dataSource != null && dataSource.Count > 0) 
  {
      dataGridView1.DataSource = dataSource; 
  }
  ```

- 第二，如果已经绑定了，可以判断当前数据集的元素个数是否为0，如果大于0则设置CurrentCell；

  ```c#
  if (dataGridView1.Rows.Count > 0) 
  { 
      dataGridView1.CurrentCell = dataGridView1.Rows[0].Cells[1]; 
  }
  ```

  顺带一提，设置时，CurrentCell对应的列，必需为可视的。



