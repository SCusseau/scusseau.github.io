---
layout: post
title: Builder Pattern with Default Values to simplify method call with default values
---

> SUMMARY
>- Simplifies method calls thus making code **more maintainable** and **readable**.
>- Ideal for handling methods with **numerous optional parameters**.
>- **Improves flexibility** without complicating the API.

Methods with many optional parameters can become cumbersome, especially when you need to set defaults for multiple options. This often leads to long, error-prone method signatures. A cleaner solution combines the **Builder Pattern** with **Default Values**, allowing you to specify only the parameters you care about and leaving the rest to defaults.

### The Problem to solve

Consider this file upload method:

```csharp
public async Task<string> UploadFileAsync(string containerName, string blobName, MemoryStream fileStream, bool overwrite = false, bool compress = false)
{
    // Logic for file upload
}
```

If you want to modify `compress`, you still have to pass values for the preceding parameters. This becomes unwieldy when there are many optional parameters.

### Solution: Combining the Builder Pattern with Default Values

We’ll use the **Builder Pattern** to manage parameters more flexibly. By grouping all parameters in a `FileUploadOptions` object and setting default values, we can configure only the parameters that need modification.

#### Step 1: Create the `FileUploadOptions` Class

Define a `FileUploadOptions` class that holds all parameters, with default values for optional ones.

```csharp
public class FileUploadOptions
{
    public string ContainerName { get; private set; }
    public string BlobName { get; private set; }
    public MemoryStream FileStream { get; private set; }
    public bool Overwrite { get; private set; } = false;  // Default
    public bool Compress { get; private set; } = false;   // Default

    private FileUploadOptions() { }

    public class Builder
    {
        private readonly FileUploadOptions _options = new FileUploadOptions();

        public Builder SetContainerName(string containerName) { _options.ContainerName = containerName; return this; }
        public Builder SetBlobName(string blobName) { _options.BlobName = blobName; return this; }
        public Builder SetFileStream(MemoryStream fileStream) { _options.FileStream = fileStream; return this; }
        public Builder SetOverwrite(bool overwrite) { _options.Overwrite = overwrite; return this; }
        public Builder SetCompress(bool compress) { _options.Compress = compress; return this; }

        public FileUploadOptions Build() => _options;
    }
}
```

#### Step 2: Use the Builder to Construct the Object

Now, use the builder to configure only the parameters you care about:

```csharp
var uploader = new FileUploader();
var uploadOptions = new FileUploadOptions.Builder()
    .SetContainerName("myContainer")
    .SetBlobName("file.txt")
    .SetFileStream(myFileStream)
    .SetOverwrite(true)  // Set only the parameters you need
    .Build();

await uploader.UploadFileAsync(uploadOptions);
```

### Key Benefits

- **Cleaner Code**: No need to specify default parameters on every method call.
- **Flexibility**: Easily modify only the parameters you care about.
- **Fluent API**: The builder pattern provides a readable, step-by-step configuration process.
- **Maintainability**: Adding new parameters becomes easier without changing method signatures.