Downloadprogress Interceptor
===================

An [OkHttp interceptor][1] which monitors download progress of HTTP response data.

[![](https://jitpack.io/v/jobinlawrance/okhttp3-downloadprogress-interceptor.svg)](https://jitpack.io/#jobinlawrance/okhttp3-downloadprogress-interceptor)

## Download  
Downloadprogress Interceptor is available on [Jitpack][2]  
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
To receive download progress updates, we subsribe to `ProgressEventBus`. Whenever a download is started by retrofit all the subscibers receives a `ProgressEvent` which has the `progress`(in percent) and `downloadIdentifier` data among others.  
Since `ProgressEventBus` monitors all the download requests with the custom header we set above, we check the `ProgressEvent.dowloadIdentfier` to filter out the `ProgressEvent` we are interested in.  
Here's an example - 
###### kotlin

```kotlin
// Inject retrofit and progressEventBus

@Inject lateinit val retrofit
@Inject lateinit val progressEventBus

val imageUrl = "https://upload.wikimedia.org/wikipedia/commons/2/24/Junonia_orithya-Thekkady-2016-12-03-001.jpg"
val imageIdentifier = "my-wiki-image"

val disposable =                                  // handle unsubscription   
                progressEventBus.observable()
                        .subscribe(
                                {
                                    if(it.downloadIdentifier == imageIdentifier) {
                                      // Display the progress here
                                      Log.d("Download Progress - ${it.progress}")
                                    }
                                },
                                {
                                    Log.d("ProgressEvent Error",it)
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

## Credits
1. [https://blog.playmoweb.com/view-download-progress-on-android-using-retrofit2-and-okhttp3-83ed704cb968](https://blog.playmoweb.com/view-download-progress-on-android-using-retrofit2-and-okhttp3-83ed704cb968)  
2. [https://github.com/square/okhttp/blob/aed222454743ebe5724d6ad438fafed37956521e/samples/guide/src/main/java/okhttp3/recipes/Progress.java](https://github.com/square/okhttp/blob/aed222454743ebe5724d6ad438fafed37956521e/samples/guide/src/main/java/okhttp3/recipes/Progress.java)  



[1]: https://github.com/square/okhttp/wiki/Interceptors
[2]: https://jitpack.io/#jobinlawrance/okhttp3-downloadprogress-interceptor
