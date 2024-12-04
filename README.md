
书接上回，前面章节已经实现Excel帮助类的第一步TableHeper的对象集合与DataTable相互转换功能，今天实现进入其第二步的核心功能ExcelHelper实现。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241203204926829-1541523313.png)


# ***01***、接口设计


下面我们根据第一章中讲解的核心设计思路，先进行接口设计，确定ExcelHelper需要哪些接口即可满足我们的要求，然后再一个一个接口实现即可。


先简单回顾一下核心设计思路，主要涉及两类操作：读和写，两种转换：DataTable与Excel转换和对象集合与Excel转换。


下面先看看设计的所有接口：



```
//根据文件路径读取Excel到DataSet
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则读取所有工作簿Sheet
public static DataSet Read(string path, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//根据文件流读取Excel到DataSet
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则读取所有工作簿Sheet
public static DataSet Read(Stream stream, string fileName, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//根据文件流读取Excel到DataSet
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则读取所有工作簿Sheet
public static DataSet Read(Stream stream, bool isXlsx, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//根据文件流读取Excel到对象集合
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则默认读取第一个工作簿Sheet
public static IEnumerable<T> Read<T>(string path, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//根据文件流读取Excel到对象集合
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则默认读取第一个工作簿Sheet
public static IEnumerable<T> Read<T>(Stream stream, string fileName, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//根据文件流读取Excel到对象集合
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则默认读取第一个工作簿Sheet
public static IEnumerable<T> Read<T>(Stream stream, bool isXlsx, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null);

//把表格数组写入Excel文件流
public static MemoryStream Write(DataTable[] dataTables, bool isXlsx, bool isColumnNameAsData);

//把表格数组写入Excel文件
public static void Write(DataTable[] dataTables, string path, bool isColumnNameAsData);

//把对象集合写入Excel文件流
public static MemoryStream Write<T>(IEnumerable models, bool isXlsx, bool isColumnNameAsData, string? sheetName = null);

//把对象集合写入Excel文件
public static void Write<T>(IEnumerable models, string path, bool isColumnNameAsData, string? sheetName = null);

```

# ***02***、根据文件路径读取Excel到DataSet


该方法是通过Excel完全路径直接读取Excel文件，因此我们首先读取到文件流，然后再调用具体处理文件流实现方法。


因为Excel中工作簿Sheet正好对应DataSet中表格DataTable，因此在不指定读取某个工作簿Sheet的情况下，默认是读取Excel中所有工作簿Sheet。


指定工作簿方式也很简单，只要传参数指定工作簿名称sheetName或者工作簿编号sheetNumber即可，提供两个参数是考虑到可能名字不好记，但是第几个工作簿Sheet会比较好记，也因此工作簿编号sheetNumber是从1开始。两者会优先处理工作簿名称sheetName。


因为表格DataTable是有列名的，通过这个列名我们可以把它和对象属性关联上，最后实现相互映射转换，而工作簿Sheet则没有这个概念，因此我们要想最终实现对象和工作簿Sheet的相互转换，就需要人为指定这样的数据。


通常的做法是以工作簿Sheet中第一行数据作为表格DataTable列名，因此我们在接口中设计了这个参数用来指定是否需要把第一行数据作为表格列名。


具体代码实现如下：



```
//根据文件路径读取Excel到DataSet
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则读取所有工作簿Sheet
public static DataSet Read(string path, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null)
{
    using var stream = new FileStream(path, FileMode.Open, FileAccess.Read);
    return Read(stream, IsXlsxFile(path), isFirstRowAsColumnName, sheetName, sheetNumber);
}

```

# ***03***、根据文件流、文件名读取Excel到DataSet


在有些场景下，不需要我们直接读取Excel文件，而是直接给一个Excel文件流。比如说文件上传，前端上传文件后，后端接收到的就是一个文件流。


同时该方法还需要传一个文件名的参数，这是因为我们Excel有两种后缀格式即“.xls”和“.xlsx”，而两种格式处理方式又不相同，因此我们需要通过名字来说识别Excel文件流的具体格式，当然如果调用方法时已经明确知道文件流是什么格式，也可以直接调用下一个重载方法。


其他参数解释上节以及详细讲解了，实现代码如下：



```
//根据文件流读取Excel到DataSet
//指定sheetName，sheetNumber则读取相应工作簿Sheet
//如果不指定则读取所有工作簿Sheet
public static DataSet Read(Stream stream, string fileName, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null)
{
    return Read(stream, IsXlsxFile(fileName), isFirstRowAsColumnName, sheetName, sheetNumber);
}

```

# ***04***、根据文件流、文件后缀读取Excel到DataSet


该方法是上面两个方法的最终实现，该方法首先会识别读取所有工作簿Sheet还是读取指定工作簿Sheet，然后调不同的方法。而两者差别也这是读一个还是读多个工作簿Sheet的差别，具体代码如下：



```
//根据文件流读取Excel到DataSet
public static DataSet Read(Stream stream, bool isXlsx, bool isFirstRowAsColumnName = false, string? sheetName = null, int? sheetNumber = null)
{
    if (sheetName == null && sheetNumber == null)
    {
        //读取所有工作簿Sheet至DataSet
        return CreateDataSetWithStreamOfSheets(stream, isXlsx, isFirstRowAsColumnName);
    }
    //读取指定工作簿Sheet至DataSet
    return CreateDataSetWithStreamOfSheet(stream, isXlsx, isFirstRowAsColumnName, sheetName, sheetNumber ?? 1);
}
//读取所有工作簿Sheet至DataSet
private static DataSet CreateDataSetWithStreamOfSheets(Stream stream, bool isXlsx, bool isFirstRowAsColumnName)
{
    //根据Excel文件后缀创建IWorkbook
    using var workbook = CreateWorkbook(isXlsx, stream);
    //根据Excel文件后缀创建公式求值器
    var evaluator = CreateFormulaEvaluator(isXlsx, workbook);
    var dataSet = new DataSet();
    for (var i = 0; i < workbook.NumberOfSheets; i++)
    {
        //获取工作簿Sheet
        var sheet = workbook.GetSheetAt(i);
        //通过工作簿Sheet创建表格
        var table = CreateDataTableBySheet(sheet, evaluator, isFirstRowAsColumnName);
        dataSet.Tables.Add(table);
    }
    return dataSet;
}
//读取指定工作簿Sheet至DataSet
private static DataSet CreateDataSetWithStreamOfSheet(Stream stream, bool isXlsx, bool isFirstRowAsColumnName, string? sheetName = null, int sheetNumber = 1)
{
    //把工作簿sheet编号转为索引
    var sheetIndex = sheetNumber - 1;
    var dataSet = new DataSet();
    if (string.IsNullOrWhiteSpace(sheetName) && sheetIndex < 0)
    {
        //工作簿sheet索引非法则返回
        return dataSet;
    }
    //根据Excel文件后缀创建IWorkbook
    using var workbook = CreateWorkbook(isXlsx, stream);
    if (string.IsNullOrWhiteSpace(sheetName) && sheetIndex >= workbook.NumberOfSheets)
    {
        //工作簿sheet索引非法则返回
        return dataSet;
    }
    //根据Excel文件后缀创建公式求值器
    var evaluator = CreateFormulaEvaluator(isXlsx, workbook);
    //优先通过工作簿名称获取工作簿sheet
    var sheet = !string.IsNullOrWhiteSpace(sheetName) ? workbook.GetSheet(sheetName) : workbook.GetSheetAt(sheetIndex);
    if (sheet != null)
    {
        //通过工作簿sheet创建表格
        var table = CreateDataTableBySheet(sheet, evaluator, isFirstRowAsColumnName);
        dataSet.Tables.Add(table);
    }
    return dataSet;
}

```

通过上图实现工作簿Sheet转换DataSet过程，可以发现大致分为三步：


第一步首先根据文件格式以及文件流获取IWorkbook；


第二步再通过文件格式以及IWorkbook获取公式求值器；


第三步再实现把工作簿Sheet转换为表格DataTable；


我们一起看看这三个代码实现：



```
//根据Excel文件后缀创建IWorkbook
private static IWorkbook CreateWorkbook(bool isXlsx, Stream? stream = null)
{
    if (stream == null)
    {
        return isXlsx ? new XSSFWorkbook() : new HSSFWorkbook();
    }
    return isXlsx ? new XSSFWorkbook(stream) : new HSSFWorkbook(stream);
}
//根据Excel文件后缀创建公式求值器
private static IFormulaEvaluator CreateFormulaEvaluator(bool isXlsx, IWorkbook workbook)
{
    return isXlsx ? new XSSFFormulaEvaluator(workbook) : new HSSFFormulaEvaluator(workbook);
}
//工作簿Sheet转换为表格DataTable
private static DataTable CreateDataTableBySheet(ISheet sheet, IFormulaEvaluator evaluator, bool isFirstRowAsColumnName)
{
    var dataTable = new DataTable(sheet.SheetName);
    //获取Sheet中最大的列数，并以此数为新的表格列数
    var maxColumnNumber = GetMaxColumnNumber(sheet);
    if (isFirstRowAsColumnName)
    {
        //如果第一行数据作为表头，则先获取第一行数据
        var firstRow = sheet.GetRow(sheet.FirstRowNum);
        for (var i = 0; i < maxColumnNumber; i++)
        {
            //尝试读取第一行每一个单元格数据，有值则作为列名，否则忽略
            string? columnName = null;
            var cell = firstRow?.GetCell(i);
            if (cell != null)
            {
                cell.SetCellType(CellType.String);
                if (cell.StringCellValue != null)
                {
                    columnName = cell.StringCellValue;
                }
            }
            dataTable.Columns.Add(columnName);
        }
    }
    else
    {
        for (var i = 0; i < maxColumnNumber; i++)
        {
            dataTable.Columns.Add();
        }
    }
    //循环处理有效行数据
    for (var i = isFirstRowAsColumnName ? sheet.FirstRowNum + 1 : sheet.FirstRowNum; i <= sheet.LastRowNum; i++)
    {
        var row = sheet.GetRow(i);
        var newRow = dataTable.NewRow();
        //通过工作簿sheet行数据填充表格新行数据
        FillDataRowBySheetRow(row, evaluator, newRow);
        //检查每单元格是否都有值
        var isNullRow = true;
        for (var j = 0; j < maxColumnNumber; j++)
        {
            isNullRow = isNullRow && newRow.IsNull(j);
        }
        if (!isNullRow)
        {
            dataTable.Rows.Add(newRow);
        }
    }
    return dataTable;
}

```

在实现工作簿Sheet转换为表格DataTable过程中，大致可以分为两步：


第一步求出工作簿Sheet中所有有效行中最宽的列编号，并以此为列数创建表格；


第二步把工作簿Sheet中所有有效行数据填充至表格中；


下面我们看看具体实现代码：



```
//获取工作簿Sheet中最大的列数
private static int GetMaxColumnNumber(ISheet sheet)
{
    var maxColumnNumber = 0;
    //在有效的行数据中
    for (var i = sheet.FirstRowNum; i <= sheet.LastRowNum; i++)
    {
        var row = sheet.GetRow(i);
        //找到最大的列编号
        if (row != null && row.LastCellNum > maxColumnNumber)
        {
            maxColumnNumber = row.LastCellNum;
        }
    }
    return maxColumnNumber;
}
//通过工作簿sheet行数据填充表格行数据
private static void FillDataRowBySheetRow(IRow row, IFormulaEvaluator evaluator, DataRow dataRow)
{
    if (row == null)
    {
        return;
    }
    for (var j = 0; j < dataRow.Table.Columns.Count; j++)
    {
        var cell = row.GetCell(j);
        if (cell != null)
        {
            switch (cell.CellType)
            {
                case CellType.Blank:
                    dataRow[j] = DBNull.Value;
                    break;
                case CellType.Boolean:
                    dataRow[j] = cell.BooleanCellValue;
                    break;
                case CellType.Numeric:
                    if (DateUtil.IsCellDateFormatted(cell))
                    {
                        dataRow[j] = cell.DateCellValue;
                    }
                    else
                    {
                        dataRow[j] = cell.NumericCellValue;
                    }
                    break;
                case CellType.String:
                    dataRow[j] = !string.IsNullOrWhiteSpace(cell.StringCellValue) ? cell.StringCellValue : DBNull.Value;
                    break;
                case CellType.Error:
                    dataRow[j] = cell.ErrorCellValue;
                    break;
                case CellType.Formula:
                    dataRow[j] = evaluator.EvaluateInCell(cell).ToString();
                    break;
                default:
                    throw new NotSupportedException("Unsupported cell type.");
            }
        }
    }
}

```

***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Ideal](https://github.com)


 本博客参考[蓝猫加速器配置下载](https://yunbeijia.com)。转载请注明出处！
