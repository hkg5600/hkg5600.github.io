
### Android Clean Architecture Sample Code

**사용기술**
- Coroutine
- Hilt
- Retrofit2

**순서** 
1. *data/api*
2. *data/datasource*
3. *data/repository*
4. *domain/repositoryImpl*
5. *domain/usecase*
6. *presentation/viewmodel*
7. *dependency injection*

### 1. data/api

> data/api/ProductApi

```kotlin
interface ProductApi {  
  @GET("some-urls-get-products")  
  suspend fun getProducts() : Response<List<ProductRemote>>  
}
```

> data/api/model/ProductRemote

```kotlin
data class ProductRemote(  
  val id: String,  
  val name: String,  
  val price: Int,  
  val createdAt: String,  
  val description: String  
)  
```

### 2. data/datasource

> data/datasource/RemoteProductDataSource

```kotlin
class RemoteProductDataSource @Inject constructor(  
  private val api: ProductApi  
) {  
  suspend fun getProducts() : List<ProductRemote> {  
	return api.getProducts().verify() //네트워크 통신의 결과를 판단.
  }  
}
```

### 3. data/repository

> data/repository/ProductRepositoryImpl

```kotlin
class ProductRepositoryImpl @Inject constructor(  
  private val dataSource: RemoteProductDataSource  
) : ProductRepository {  
  
  override suspend fun getProducts(): List<Product> {  
    return dataSource.getProducts().toDomainModel()  
  }
}
```

> data/api/mapper/ProductRemoteMapper.kt 또는 data/api/model/ProductRemote의 내부

```kotlin
fun List<ProductRemote>.toDomainModel() : List<Product> {  
  return this.map {  
    it.toDomainModel()  
  }  
}

fun ProductRemote.toDomainModel() : Product {  
  return Product(  
    id = this.id,  
    name = this.name,  
    price = this.price,  
    createdAt = formatDate(this.createdAt), //datetime 포맷 변경
    description = this.description  
  )  
}
```

### 4. domain/repository

> domain/repository/ProductRepository

```kotlin
interface ProductRepository {  
  suspend fun getProducts() : List<Product>  
}
```

### 5. domain/usecase

> domain/usecase/GetProductsUseCase

```kotlin
class GetProductsUseCase @Inject constructor(  
  private val repository: ProductRepository,  
  @IoDispatcher coroutineDispatcher: CoroutineDispatcher  
) : CoroutineUseCase<Unit, List<Product>>(coroutineDispatcher) {  

  override suspend fun execute(parameter: Unit): List<Product> {  
    return repository.getProducts()  
  }
  
}
```

> domain/usecase/utils/CoroutineUseCase

```kotlin
abstract class CoroutineUseCase<P, R>(private val coroutineDispatcher: CoroutineDispatcher) {  
  
  protected abstract suspend fun execute(parameter: P) : R  
  
  suspend operator fun invoke(parameter: P) : Result<R> {  
    return try {  
      withContext(coroutineDispatcher) {  
        val result = execute(parameter)  
        Result.Success(result)  
      }  
    } catch (e: Exception) {  
      Result.Error(e)  
    } 
  }
}
```

> domain/usecase/utils/Result

```kotlin
sealed class Result<out T> {  
  data class Success<T>(val data: T) : Result<T>()  
  data class Error(val exception: Exception) : Result<Nothing>()  
}  
  
val Result<*>.succeeded  
  get() = this is Result.Success && data != null  
  
val <T> Result<T>.data //Success일 경우에만 사용하도록 주의
  get() = (this as? Result.Success)?.data ?: throw Exception("Result data is null")
```

### 6. presentation/viewmodel

> presentation/main/MainViewModel

```kotlin
class MainViewModel @ViewModelInject constructor(  
  private val getProductsUseCase: GetProductsUseCase  
) : ViewModel() {  
  
  private val _products = MutableLiveData<List<Product>>()  
  val product : LiveData<List<Product>>  
    get() = _products  
  
  init {  
    getProducts()  
  }  
  
  fun getProducts() {  
    viewModelScope.launch {  
    
      val result = getProductsUseCase(Unit)  
      if (result.succeeded) {  
        _products.value = result.data  
      } else {  
        //handle error  
      }  
    }  
  }  
}
```

### 7. dependency injection

> di/RetrofitModule

```kotlin
@Module  
@InstallIn(ApplicationComponent::class)  
object RetrofitModule {  
  
  @Provides  
  fun provideProductApi() : ProductApi = Retrofit.Builder()  
    .baseUrl("https://some-base-url")  
    .addConverterFactory(GsonConverterFactory.create())  
    .build()  
    .create(ProductApi::class.java)  
}
```

> di/RepositoryModule

```kotlin
@Module  
@InstallIn(ApplicationComponent::class)  
object RepositoryModule {  
  
  @Singleton  
  @Provides  
  fun provideProductRepository(  
    remoteProductDataSource: RemoteProductDataSource  
  ) : ProductRepository {  
    return ProductRepositoryImpl(remoteProductDataSource)  
  }  
  
}
```

> di/IoDispatcher

```kotlin
annotation class IoDispatcher
```

> di/CoroutineModule

```kotlin
@InstallIn(ApplicationComponent::class)  
@Module  
object CoroutinesModule {  
  
  @IoDispatcher  
  @Provides  
  fun providesIoDispatcher(): CoroutineDispatcher = Dispatchers.IO  
      
}
```
