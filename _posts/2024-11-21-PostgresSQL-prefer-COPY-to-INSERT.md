---
layout: post
title: Why You Should Prefer COPY Over INSERT for Large Datasets in PostgreSQL
---

> Summary
> 1. For large datasets `COPY` command in PostgreSQL with `Npgsql` is significantly more efficient than executing multiple individual `INSERT` statements 
> 2. The `COPY` command operates within a single transaction by default, ensuring that all data is either inserted or rolled back if an error occurs.

---

### **Why `COPY` is More Efficient**

1. **Reduced Round-Trips**:
    - With individual `INSERT` statements, each row requires a separate round-trip to the database.
    - The `COPY` command sends all data in bulk, reducing the number of round-trips.
      
2. **Optimized for Bulk Operations**:
    - `COPY` is designed specifically for bulk data transfers, minimizing overhead compared to multiple `INSERT` commands.
      
3. **Binary Format Support**:
    - When using `COPY` with binary format, the data is transmitted more compactly and parsed faster than plain text.
      
4. **Transactional Integrity**:
    - The `COPY` command operates within a single transaction by default, ensuring that all data is either inserted or rolled back if an error occurs.

---

### **Detailed Implementation with `Npgsql`**

#### **Use Case: Migrating Data from SQL Server to PostgreSQL**

Goal : transferring all data from a SQL Server table to a PostgreSQL table while maintaining data integrity and consistency. 

1. **Data Retrieval from SQL Server**:
    - Execute a `SELECT` statement on the source SQL Server table to fetch all records.
    - Map the retrieved data to a `List<MyDTO>`, where `MyDTO` 
2. **Data Insertion into PostgreSQL** → Two options 

- Option 1 : Using Batch Inserts 
```SQL
INSERT INTO my_table (column1, column2, column3) VALUES (@value1, @value2, @value3);
```
- Option 2 : Using `COPY` Command for Bulk Inserts
```SQL
COPY my_table (column1, column2, column3) FROM STDIN (FORMAT BINARY)
```

#### Example

Using this DTO

```csharp
public class MyDto
{
    public string? Column1 { get; set; }
    public int? Column2 { get; set; }
    public DateTime? Column3 { get; set; }
}
```

Create the Bulk Insert Method

```csharp
using Npgsql;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public async Task BulkInsertDtoListAsync(List<MyDto> dtoList, string connectionString)
{
    await using var connection = new NpgsqlConnection(connectionString);
    await connection.OpenAsync();

    // Start the binary COPY command
    await using var writer = connection.BeginBinaryImport(@"
        COPY my_table (column1, column2, column3) FROM STDIN (FORMAT BINARY)");

    foreach (var dto in dtoList)
    {
        await writer.StartRowAsync();

        // Write each column value
        await writer.WriteAsync(dto.Column1 ?? DBNull.Value); // Handles null values
        await writer.WriteAsync(dto.Column2 ?? (object)DBNull.Value);
        await writer.WriteAsync(dto.Column3 ?? (object)DBNull.Value);
    }

    // Complete the COPY operation
    await writer.CompleteAsync();
}
```

*Note:*

- **`BeginBinaryImport`**: Initializes the `COPY` command in binary mode.
- **`StartRowAsync`**: Indicates the beginning of a new row.
- **`WriteAsync`**: Writes values for each column in a row. Handles nulls using `DBNull.Value`.
- **`CompleteAsync`**: Finalizes the operation, committing all rows in bulk.

---
### **Performance Benchmark**

- **Single Inserts**: If you insert 10,000 rows using individual `INSERT` commands, you’ll make 10,000 network calls to the database.
- **`COPY` Command**: Inserts all rows in a single network call. This reduces the overhead and speeds up data loading by several orders of magnitude, especially for large datasets.

---

### **Error Handling and Transactions**

1. **Error Handling**:
    - Wrap the code in `try-catch` to handle exceptions such as connection issues or data format mismatches.
2. **Transactions**:
    - Use PostgreSQL transactions to ensure all rows are inserted atomically:
        
        ```csharp
        await using var transaction = await connection.BeginTransactionAsync();
        try
        {
            await BulkInsertDtoListAsync(dtoList, connectionString);
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
        ```
        

---

### **When to Use `COPY`**

- **Large Data Sets**: Use for thousands or millions of rows.
- **Frequent Bulk Operations**: If you regularly need to insert large amounts of data.
- **Time-Sensitive Operations**: When performance is a priority.

