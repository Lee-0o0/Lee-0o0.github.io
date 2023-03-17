---
title: RDLC报表打印工具类
date: 2023-03-17 10:23:18
tags: ["C#","WinForm", "IT技术","RDLC"]
categories: ["IT技术"]
---

不预览直接打印RDLC报表工具类：

```c#
public class ReportPrintHelper
{
    public static bool ExportEMF = false;

    private LocalReport report;

    private int m_currentPageIndex;
    private IList<Stream> m_streams;

    public ReportPrintHelper()
    {
        report = new LocalReport();
    }


    /// <summary>
    /// 将指定路径的报表打印
    /// </summary>
    /// <param name="printerName">打印机名称</param>
    /// <param name="reportPath">报表路径，以当前程序所在位置为基本路径</param>
    /// <param name="deviceInfo">打印纸信息，可通过GetDeviceInfoUnitCM()方法获取</param>
    /// <param name="dataSource">报表数据集</param>
    /// <param name="reportParams">报表参数</param>
    public void Run(string printerName, string reportPath, string deviceInfo, Dictionary<string, object> dataSource, Dictionary<string, object> reportParams)
    {
        if (string.IsNullOrEmpty(printerName))
            throw new Exception("Error: printer name is empty.");
        if (string.IsNullOrEmpty(reportPath))
            throw new Exception("Error:report path is empty.");
        if (string.IsNullOrEmpty(deviceInfo))
            throw new Exception("Error:device info is empty");

        report.ReportPath = AppDomain.CurrentDomain.BaseDirectory + reportPath;

        if (dataSource != null)
        {
            foreach (KeyValuePair<string, object> keyValue in dataSource)
            {
                report.DataSources.Add(new ReportDataSource(keyValue.Key, keyValue.Value));
            }
        }
        if (reportParams != null)
        {
            foreach (KeyValuePair<string, object> keyValue in reportParams)
            {
                report.SetParameters(new ReportParameter(keyValue.Key, keyValue.Value.ToString()));
            }
        }

        Export(report, deviceInfo);
        Print(printerName);
        Dispose();
    }


    /// <summary>
    /// 获取打印纸信息
    /// </summary>
    /// <param name="width">打印纸的宽（以厘米为单位）</param>
    /// <param name="height">打印纸的高（以厘米为单位）</param>
    /// <param name="marginLeft">打印纸的左边距（以厘米为单位）</param>
    /// <param name="marginRight">打印纸的右边距（以厘米为单位）</param>
    /// <param name="marginTop">打印纸的上边距（以厘米为单位）</param>
    /// <param name="marginBottom">打印纸的下边距（以厘米为单位）</param>
    /// <returns></returns>
    public static string GetDeviceInfoUnitCM(double width,
                                             double height,
                                             double marginLeft,
                                             double marginRight,
                                             double marginTop,
                                             double marginBottom)
    {
        return $@"<DeviceInfo>
                        <OutputFormat>EMF</OutputFormat>
                        <PageWidth>{width}cm</PageWidth>
                        <PageHeight>{height}cm</PageHeight>
                        <MarginTop>{marginTop}cm</MarginTop>
                        <MarginLeft>{marginLeft}cm</MarginLeft>
                        <MarginRight>{marginRight}cm</MarginRight>
                        <MarginBottom>{marginBottom}cm</MarginBottom>
                    </DeviceInfo>";
    }


    private Stream CreateStream(string name,
                                string fileNameExtension,
                                Encoding encoding,
                                string mimeType,
                                bool willSeek)
    {
        Stream stream;
        if (ExportEMF)
        {
            // 以下四行代码将打印的内容导出为图片
            string path = Environment.CurrentDirectory + "\\EMFCache\\";
            if (!Directory.Exists(path))
                Directory.CreateDirectory(path);
            stream = new FileStream(path + name + DateTime.Now.Millisecond + "." + fileNameExtension, FileMode.Create);
        }
        else
        {
            stream = new MemoryStream();
        }
        m_streams.Add(stream);
        return stream;
    }


    private void Export(LocalReport report, string deviceInfo)
    {
        Warning[] warnings;
        m_streams = new List<Stream>();
        report.Render("Image", deviceInfo, CreateStream, out warnings);

        foreach (Stream stream in m_streams)
            stream.Position = 0;
    }


    private void PrintPage(object sender, PrintPageEventArgs ev)
    {
        Metafile pageImage = new Metafile(m_streams[m_currentPageIndex]);

        // Adjust rectangular area with printer margins.
        Rectangle adjustedRect = new Rectangle(ev.PageBounds.Left - (int)ev.PageSettings.HardMarginX,
                                               ev.PageBounds.Top - (int)ev.PageSettings.HardMarginY,
                                               ev.PageBounds.Width,
                                               ev.PageBounds.Height);


        ev.Graphics.FillRectangle(Brushes.White, adjustedRect);

        // Draw the report content
        ev.Graphics.DrawImage(pageImage, adjustedRect);

        // Prepare for the next page. Make sure we haven't hit the end.
        m_currentPageIndex++;
        ev.HasMorePages = (m_currentPageIndex < m_streams.Count);
    }

    private void Print(string printerName)
    {
        if (m_streams == null || m_streams.Count == 0)
            throw new Exception("Error: no stream to print.");

        PrintDocument printDoc = new PrintDocument();
        if (!printDoc.PrinterSettings.IsValid)
        {
            throw new Exception("Error: cannot find the default printer.");
        }

        printDoc.PrinterSettings.PrinterName = printerName;
        printDoc.PrintPage += new PrintPageEventHandler(PrintPage);
        m_currentPageIndex = 0;
        printDoc.Print();

    }

    private void Dispose()
    {
        if (m_streams != null)
        {
            foreach (Stream stream in m_streams)
            {
                stream.Close();
            }
            m_streams = null;
        }
    }

}
```

注意点：

- 我们设计的报表的宽高加上`GetDeviceInfoUnitCM()`方法中设定的边距不可超过`GetDeviceInfoUnitCM()`方法中设定的宽高，否则会出现分页现象。

  ![image-20230317102840091](/img/RDLC%E6%8A%A5%E8%A1%A8%E6%89%93%E5%8D%B0%E5%B7%A5%E5%85%B7%E7%B1%BB/image-20230317102840091.png)

- 使用方法：

  ```c#
  ReportPrintHelper reportPrintHelper = new ReportPrintHelper();
  string deviceInfo = ReportPrintHelper.GetDeviceInfoUnitCM(29.7, 21, 0, 2, 0, 0);
  reportPrintHelper.Run("Foxit PDF Reader Printer", "/Report/DemoReport.rdlc", deviceInfo, null, null);
  ```

  打印效果：

  ![image-20230317103228483](/img/RDLC%E6%8A%A5%E8%A1%A8%E6%89%93%E5%8D%B0%E5%B7%A5%E5%85%B7%E7%B1%BB/image-20230317103228483.png)

  可以看到，如果我们设计的报表没有完整填充打印纸，则页眉、主体和页脚部分都会进行延申填充。所以在设计报表时，最好能完全填充打印纸大小。