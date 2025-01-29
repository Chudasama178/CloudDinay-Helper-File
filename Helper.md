 # Cloudianry use to manage images
 
 This demonstrates how to use Cloudinary for image store and use in project

 ---
 ### 1. Install Required Packeges

 Run the following commands to install necessary packages:

 ```bash
dotnet add package CloudinaryDotNet
```

### 2.Update `appsettings.json`

Create a configuration section in your appsettings.json file for your Cloudinary credentials: 

```csharp
{
  "CloudinarySettings": {
    "CloudName": "your-cloud-name",
    "ApiKey": "your-api-key",
    "ApiSecret": "your-api-secret"
  }
}

```


### 3. Update `Program.cs`

#### a. Register the settings and the Cloudinary service in the  Program.cs file:

```csharp
builder.Services.AddSingleton(x =>
{
    var cloudinarySettings = builder.Configuration.GetSection("CloudinarySettings").Get<CloudinarySettings>();
    var account = new Account(
        cloudinarySettings.CloudName,
        cloudinarySettings.ApiKey,
        cloudinarySettings.ApiSecret
    );
    return new Cloudinary(account);
});
```

#### b.Create a CloudinarySettings class to bind your configuration:

```csharp
public class CloudinarySettings
{
    public string CloudName { get; set; }
    public string ApiKey { get; set; }
    public string ApiSecret { get; set; }
}
```

### 4.Defines `Models`

Create a `ProductModel` class into Models folder :

```csharp
public class ProductModel
{
    public int ProductID { get; set; }
    public string Name { get; set; }
    public string ImageUrl { get; set; }
}
public class ProductInsertUpdateModel
{
    public int? ProductID { get; set; }
    public string Name { get; set; }
    public IFormFile ImageFile { get; set; }
}
public class ApiResponse
{
    public bool Success { get; set; }
    public string Message { get; set; }
}
```

### 5. Implement Cloudinary `SelectByPk`,`Insert`,`Update`,`Delete` opration in `ProdductRepository`

Create `ProductRepository` class into data folder

```csharp
public interface IProductRepository
{
    Task<ProductModel> GetProductByIdAsync(int productId);
    Task<ApiResponse> AddProductAsync(ProductInsertUpdateModel product);

    Task<ApiResponse> UpdateProductAsync(ProductInsertUpdateModel product, int ProductID);
    Task<bool> DeleteProductAsync(int productId);
    Task<string> UploadImageAsync(IFormFile file);
}

public class ProductRepo : IProductRepository
{
    #region  Configuration
    private readonly string _connectionStr;
    private readonly Cloudinary _cloudinary;

    public ProductRepo(IConfiguration configuration)
    {
        _connectionStr = configuration.GetConnectionString("ConnectionString");

        var cloudinarySettings = configuration.GetSection("CloudinarySettings");
        var account = new Account(
            cloudinarySettings["CloudName"],
            cloudinarySettings["ApiKey"],
            cloudinarySettings["ApiSecret"]
        );

        _cloudinary = new Cloudinary(account);
    }
    #endregion

    #region SelectByPk
    public async Task<ProductModel> GetProductByIdAsync(int productId)
    {
        ProductModel product = null;
        using (SqlConnection conn = new SqlConnection(_connectionStr))
        {
            SqlCommand cmd = new SqlCommand("PR_Product_SelectByPk", conn)
            {
                CommandType = CommandType.StoredProcedure
            };
            conn.Open();
            cmd.Parameters.AddWithValue("@ProductID", productId);
            SqlDataReader reader = cmd.ExecuteReader();
            if (reader.Read())
            {
                product = new ProductModel()
                {
                    ProductID = Convert.ToInt32(reader["ProductID"]),
                    Name = reader["Name"].ToString(),
                    ImageUrl = reader["ImageUrl"].ToString(),
                };
            }
            reader.Close();
        }
        return product;
    }
    #endregion

    #region Insert
    public async Task<ApiResponse> AddProductAsync(ProductInsertUpdateModel product)
    {
        string imageUrl = null;

        if (product.ImageFile != null)
        {
            imageUrl = await UploadImageAsync(product.ImageFile); 
        }

        using (SqlConnection conn = new SqlConnection(_connectionStr))
        {
            SqlCommand cmd = new SqlCommand("Pr_Product_Insert", conn)
            {
                CommandType = CommandType.StoredProcedure
            };
            conn.Open();
            // Add parameters to the SQL command
            cmd.Parameters.AddWithValue("@Name", product.Name);
            cmd.Parameters.AddWithValue("@ImageUrl", imageUrl ?? (object)DBNull.Value);
            try
            {
                int result = cmd.ExecuteNonQuery();
                if (result > 0)
                {
                    return new ApiResponse
                    {
                        Success = true,
                        Message = "Product inserted successfully."
                    };
                }

                return new ApiResponse
                {
                    Success = false,
                    Message = "Failed to insert product."
                };
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error during product insertion: {ex.Message}");
                return new ApiResponse
                {
                    Success = false,
                    Message = "Failed to insert product."
                };
            }
        }
    }
    #endregion

    #region Update
    public async Task<ApiResponse> UpdateProductAsync(ProductInsertUpdateModel product, int ProductID)
    {
        string imageUrl = null;

        if (product.ImageFile != null)
        {
            imageUrl = await UploadImageAsync(product.ImageFile); 
        }

        using (SqlConnection conn = new SqlConnection(_connectionStr))
        {
            SqlCommand cmd = new SqlCommand("Pr_Product_Update", conn)
            {
                CommandType = CommandType.StoredProcedure
            };
            conn.Open();

            cmd.Parameters.AddWithValue("@ProductID", ProductID);
            cmd.Parameters.AddWithValue("@Name", product.Name);
            cmd.Parameters.AddWithValue("@ImageUrl", imageUrl ?? (object)DBNull.Value);

            try
            {
                int result = cmd.ExecuteNonQuery(); 
                Console.WriteLine($"Rows affected: {result}");
                if (result >= 0)  
                {
                    return new ApiResponse
                    {
                        Success = true,
                        Message = "Product updated successfully."
                    };
                }

                return new ApiResponse
                {
                    Success = false,
                    Message = "Failed to update product."
                };
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error during product update: {ex.Message}");
                return new ApiResponse
                {
                    Success = false,
                    Message = "Failed to update product."
                };
            }
        }
    }

    #endregion

    #region Delete
    public async Task<bool> DeleteProductAsync(int productId)
    {
        var product = await GetProductByIdAsync(productId);
        if (product == null)
        {
            return false;
        }

        if (!string.IsNullOrEmpty(product.ImageUrl))
        {
            var imagePublicId = product.ImageUrl.Split('/').Last().Split('.').First();
            var deletionParams = new DeletionParams(imagePublicId);
            var deletionResult = await _cloudinary.DestroyAsync(deletionParams);

            if (deletionResult.StatusCode != System.Net.HttpStatusCode.OK)
            {
                Console.WriteLine("Failed to delete image from Cloudinary.");
                return false;
            }
        }

        using (SqlConnection conn = new SqlConnection(_connectionStr))
        {
            SqlCommand cmd = new SqlCommand("PR_Product_Delete", conn)
            {
                CommandType = CommandType.StoredProcedure
            };
            conn.Open();
            cmd.Parameters.AddWithValue("@ProductID", productId);
            int rowAffected = cmd.ExecuteNonQuery();
            return rowAffected > 0;
        }
    }

    #endregion

    #region Upload Image to Cloudinary
    public async Task<string> UploadImageAsync(IFormFile file)
    {
        if (file == null)
        {
            throw new ArgumentNullException(nameof(file), "File cannot be null.");
        }

        var uploadParams = new ImageUploadParams()
        {
            File = new FileDescription(file.FileName, file.OpenReadStream())
        };

        var uploadResult = await _cloudinary.UploadAsync(uploadParams);

        if (uploadResult.StatusCode != System.Net.HttpStatusCode.OK)
        {
            throw new Exception("Failed to upload image to Cloudinary.");
        }

        return uploadResult?.SecureUrl?.ToString();
    }
    #endregion
}
```

### 6. Implement Cloudinary `SelectByPk`,`Insert`,`Update`,`Delete` opration in `ProductController`

```csharp
    [Route("api/[controller]")]
[ApiController]
public class ProductController : ControllerBase
{
    #region Fields & Constructor
    private readonly IProductRepository _productRepository;

    public ProductController(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }
    #endregion

    #region SelectByPk-Product
    [HttpGet("SelectByPk/{productId}")]
    public async Task<IActionResult> GetProductById(int productId)
    {
        var product = await _productRepository.GetProductByIdAsync(productId);
        if (product == null)
            return NotFound();

        return Ok(product);
    }
    #endregion

    #region Insert-Product
    [HttpPost("Insert")]
    public async Task<IActionResult> AddProduct([FromForm] ProductInsertUpdateModel product)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var response = await _productRepository.AddProductAsync(product);

        if (!response.Success)
        {
            return BadRequest(response);
        }
        return Ok(response);
    }
    #endregion

    #region Update
    [HttpPut("Update/{ProductID}")]
    public async Task<IActionResult> UpdateProduct([FromForm] ProductInsertUpdateModel product, int ProductID)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var response = await _productRepository.UpdateProductAsync(product, ProductID);
        if (!response.Success)
        {
            return BadRequest(response);
        }

        return Ok(response);
    }
    #endregion

    #region Delete-Product
    [HttpDelete("Delete/{productId}")]
    public async Task<IActionResult> DeleteProduct(int productId)
    {
        try
        {
            var deleted = await _productRepository.DeleteProductAsync(productId);
            if (!deleted)
                return NotFound();

            return Ok(new { success = true, message = "Product deleted successfully." });
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Internal server error: {ex.Message}");
        }
    }
    #endregion
}
```


