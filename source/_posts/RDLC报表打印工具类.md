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
    public static bool ExportEMF = true;
    private LocalReport report;
    private bool landscape;

    private int m_currentPageIndex;
    private IList<Stream> m_streams;

    private string paperName = "";
    PrintDocument printDoc;

    public ReportPrintHelper()
    {
        report = new LocalReport();
        printDoc = new PrintDocument();
    }

    public void Run(string paperName, 
                    string printerName, 
                    string reportPath, 
                    string deviceInfo, 
                    Dictionary<string, object> dataSource, 
                    Dictionary<string, object> reportParams, 
                    bool isLandscape = false)
    {
        if (string.IsNullOrEmpty(printerName))
            throw new Exception("Error: printer name is empty.");
        if (string.IsNullOrEmpty(reportPath))
            throw new Exception("Error:report path is empty.");
        if (string.IsNullOrEmpty(deviceInfo))
            throw new Exception("Error:device info is empty");

        this.paperName = "A4";
        if (!string.IsNullOrEmpty(paperName))
            this.paperName = paperName;

        this.landscape = isLandscape;
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


    public static string GetDeviceInfoUnitCM(double width,
                                             double height,
                                             double marginLeft,
                                             double marginRight,
                                             double marginTop,
                                             double marginBottom)
    {
        return $@"<DeviceInfo>
                        <OutputFormat>PNG</OutputFormat>
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
        //Metafile pageImage = new Metafile(m_streams[m_currentPageIndex]);
        Bitmap pageImage = new Bitmap(m_streams[m_currentPageIndex]);
        // Adjust rectangular area with printer margins.
        Rectangle adjustedRect = new Rectangle(0,
                                               0,
                                               ev.PageSettings.Bounds.Width,
                                               ev.PageSettings.Bounds.Height);

        if (landscape)
            pageImage.RotateFlip(RotateFlipType.Rotate90FlipNone);

        ev.Graphics.FillRectangle(Brushes.White, adjustedRect);

        // Draw the report content
        ev.Graphics.DrawImage(pageImage, adjustedRect);
        //ev.Graphics.DrawRectangle(Pens.Red, adjustedRect);

        // Prepare for the next page. Make sure we haven't hit the end.
        m_currentPageIndex++;
        ev.HasMorePages = (m_currentPageIndex < m_streams.Count);
    }

    private void Print(string printerName)
    {
        if (m_streams == null || m_streams.Count == 0)
            throw new Exception("Error: no stream to print.");

        if (!printDoc.PrinterSettings.IsValid)
        {
            throw new Exception("Error: cannot find the default printer.");
        }

        printDoc.PrinterSettings.PrinterName = printerName;
        foreach(PaperSize paper in printDoc.PrinterSettings.PaperSizes)
        {
            if (this.paperName.Equals(paper.PaperName))
            {
                printDoc.DefaultPageSettings.PaperSize = paper;
                printDoc.PrinterSettings.DefaultPageSettings.PaperSize = paper;
            }
        }

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
  reportPrintHelper.Run(null, "Foxit PDF Reader Printer", "/Report/DemoReport.rdlc", deviceInfo, null, null);
  ```

  打印效果：

  ![image-20230317103228483](/img/RDLC%E6%8A%A5%E8%A1%A8%E6%89%93%E5%8D%B0%E5%B7%A5%E5%85%B7%E7%B1%BB/image-20230317103228483.png)

  可以看到，如果我们设计的报表没有完整填充打印纸，则页眉、主体和页脚部分都会进行延申填充。所以在设计报表时，最好能完全填充打印纸大小。