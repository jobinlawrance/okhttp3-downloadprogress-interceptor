Download Progress Interceptor
===================

An [OkHttp interceptor][1] which monitors download progress of HTTP response data.

[![](https://jitpack.io/v/jobinlawrance/okhttp3-downloadprogress-interceptor.svg)](https://jitpack.io/#jobinlawrance/okhttp3-downloadprogress-interceptor)

## Download  
downloadprogress-interceptor is available on [Jitpack][2]  
add the JitPack repository in your project build.gradle (in top level dir):
```gradle
repositories {
    jcenter()
    maven { url "https://jitpack.io" }
}
```
and the dependency in the build.gradle of the module:  
```gradle
dependencies {
    compile 'com.github.jobinlawrance:okhttp3-downloadprogress-interceptor:1.0.1'
}
```  

## Usage  

### Prerequisite  
The progress monitoring won't work if the reponse doesn't have a `content-length` in it's header.  

### Creating the interceptor  
###### kotlin  
We use a simple event bus `ProgressEventBus` to track the progress of all the download requests.  
Since `OkHttpClient` and `Retrofit` is instantiated as application wise singletons, the same should be done with `ProgressEventBus` and it should be injected to the classes that requires the progress monitoring.
```kotlin
val progressEventBus: ProgressEventBus = ProgressEventBus()
val downloadInterceptor: DownloadProgressInterceptor = DownloadProgressInterceptor(progressEventBus)

val okHttpClient = OkHttpClient.Builder()
                      .addNetworkInterceptor(downloadInterceptor)
                      .build()

val retrofit = Retrofit.Builder()
               ...                // base url, adapter factory etc
               .client(okHttpClient)
               .build();
                     
```

### Custom Request Header
Since `OkHttp` client is used for all Http requests, in order to differentiate the request we are interested in, we add a custom header to it.  
Our retrofit interface will look like this -
###### kotlin
```kotlin
interface PhotoService {
  @GET
  @Streaming
  fun downloadPhoto(@Url url: String, @Header(DOWNLOAD_IDENTIFIER_HEADER) identifier: String) : Observable<ResponseBody>
}
```  

### Downloading and checking progress
```kotlin
// Inject retrofit and progressEventBus

@Inject lateinit val retrofit
@Inject lateinit val progressEventBus

val imageUrl = "https://upload.wikimedia.org/wikipedia/commons/2/24/Junonia_orithya-Thekkady-2016-12-03-001.jpg"
val imageIdentifier = "my-wiki-image"

val disposable =                                
                progressEventBus.observable()
                        .subscribe(
                                {
                                    if(it.downloadIdentifier == imageIdentifier) {
                                      // Display the progress here
                                      Log.d("Download Progress - ${it.progress}")
                                    }
                                },
                                {
                                    Timber.e(it, "ProgressEvent Error")
                                },
                                {
                                    Timber.d("ProgressEvent Complete")
                                }
                        )

retrofit.create(PhotoService::class.java)
                .downloadPhoto(imageUrl, imageIdentifier)
                .subscribe(
                        {   
                            // onNext(resonseBody)
                            // Download start, 
                            // code to save the responseBody to file in storage
                            
                        },
                        {   
                            // onError()
                            // Error in dowloading
                        },
                        {
                           // Download complete
                           if (!disposable.isDisposed) {
                                disposable.dispose()
                            }
                        }
                )
    
```



[1]: https://github.com/square/okhttp/wiki/Interceptors
[2]: https://jitpack.io/#jobinlawrance/okhttp3-downloadprogress-interceptor
