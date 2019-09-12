# Retrofit2.1
android Retrofit2.1源码解析

## <font color =red>一、Retrofit.java</font>
*   整个类采用构造者模式
### 1、主要成员包括：
#### 1.1、okhttp3.Call.Factory
##### 1.1.1、子类包含：okHttpClient
##### 1.1.2、默认值为：new OkHttpClient();
```java
            if(callFactory==null){
               callFactory=new OkHttpClient();
            }
```
#### 1.2、CallAdapter.Factory---List
* 支持设置serviceMethod的返回值为非Call的其他类型
* 例如：Call<BaseResult<String>> ---->Observable<BaseResult<String>>
##### 1.2.1、数组中添加默认值ExcutorCallAdapterFactory
```java
            List<CallAdapter.Factory> adapterFactories= new ArrayList(this.adapterFactories);
            adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
```
#### 1.3、Converter.Factory---List
*  解析实体类功能的工厂类，例如把json数据转为Object
##### 1.3.1、无默认值，所有必须手动设置

#### 1.4、Executor
* callBack回调时执行所在的线程
##### 1.4.1、platform平台为android,默认值为MainThreadExecutor
```java
            public Platform findPlatform(){
                try{
                    Class.forName("android.os.Build");
                    return Android();
                }catch(ClassNotFoundException e){
                }
            }

            public class Android extends Platform{
                public Executor defaultCallbackExecutor(){
                     return new MainThreadExecutor();
                }
            }
```
#### 1.5、serviceMethodCache---LinkedHashMap<Method,ServiceMethod>
#### 1.6、validaEagerly
* 一个boolean类型的flag，用于是否提前验证所有的serviceMethod参数是否符合规则
* 逻辑在create()方法中
```java
            if(validateEagerly)(
            )
```
#### 1.7、HttpUrl
* baseUrl的包装类，会有一个验证功能，验证格式是否符合

### 2、动态代理主要方法
```java
      public <T> T create(final Class<T> service){
          Utils.validateServiceInterface(service);//验证是否是接口
          if(validateEagerly){
              //一次性验证所有方法参数的合法性
              for(Method method : service.getDeclaredMethods()){
                  loadServiceMethod(method);
              }
          }
          //动态代理的主要方法，service的任何方法调用都会被拦截这里处理
          return (T)Proxy.newProxyInstance(service.getClassLoader(),new Class<?>[]{service},
              new InvocationHandler(){
                  public Object invoke(Object proxy,Method method,Object[] args) throws Throwable{
                       //对method参数进行解析并保存到serviceMethodCache中
                       ServiceMethod serviceMethod=loadServiceMethod(method);
                       OkHttpCall okHttpCall=new OkHttpCall<>(serviceMethod,args);
                       return serviceMethod.callAdapter.adapter(okHttpCall);
                  }
              }
          );
      }

      ServiceMethod loadServiceMethod(Method method){
           ServiceMethod result;
           synchronized(serviceMethodCache){
                result=serviceMethodCache.get(method);
                if(result==null){
                    result=new ServiceMethod.Builder(this,method).build;
                    serviceMethodCache.put(method,result);
                }
           }
           return result;
      }
```
## <font color =red>二、ServiceMethod.java</font>
* 主要用反射与注解获取实际参数类型
### 1、主要成员有：
#### 1.1、ParameterHandler<?>[] parameterHandlers;
* 参数解析出数据后，每个annotation对应一个ParameterHandler,提供相应的方法用于构建
            requestBuilder
#### 1.2、Converter<ResponseBody,T> responseConverter;
* 根据设置的Converter.Factory生成responseConverter
#### 1.3、CallAdapter<?> callAdapter;
* 根据设置的CallAdapter.Factory构造callAdapter的返回参数,默认值DefaultCallAdapterFactory
```java
            return new CallAdapter<Call<?>>() {
                  @Override public Type responseType() {
                    return responseType;
                  }

                  @Override public <R> Call<R> adapt(Call<R> call) {
                    return call;
                  }
                };
```
### 2、伪代码流程
```java
       ServiceMethod.Builder(Retrofit retrofit,Method method){
           this.methodAnnotations = method.getAnnotations();
           this.parameterTypes = method.getGenericParameterTypes();
           this.parameterAnnotationsArray = method.getParameterAnnotations();
       }.build(){
           callAdapter = createCallAdapter();
           responseConverter = createResponseConverter();
           for(Annotation annotation : methodAnnotations) {
               if (annotation instanceof DELETE) {
                   parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false(boolean hasBody)){
                       this.httpMethod = httpMethod;
                       this.hasBody = hasBody;
                       this.relativeUrl = value;
                       this.relativeUrlParamNames = parsePathParameters(value);
                   };
               } else if (annotation instanceof GET) {
                 ...
               } else if (annotation instanceof HEAD) {
                 ...
               } else if(annotation instanceof retrofit2.http.Headers){
                 parseHeaders(String[] headers);
               }
               ...
           }
           int parameterCount = parameterAnnotationsArray.length;
           parameterHandlers = new ParameterHandler<?>[parameterCount];
           for (int p = 0; p < parameterCount; p++) {
                   Type parameterType = parameterTypes[p];
                   Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
                   parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations){
                        for (Annotation annotation : annotations) {
                             if (annotation instanceof Url) {
                                 return new ParameterHandler.RelativeUrl(){
                                     @Override void apply(RequestBuilder builder, Object value) {
                                       builder.setRelativeUrl(value);
                                     }
                                 };
                             }else if(annotation instanceof Path){
                                 return new ParameterHandler.Path<>(name, converter, path.encoded()){
                                     @Override void apply(RequestBuilder builder, T value) throws IOException {
                                       builder.addPathParam(name, valueConverter.convert(value), encoded);
                                     }
                                 };
                             }else if (annotation instanceof Query){

                             }
                             result = annotationAction;
                        }
                        return result;
                   };
           }
           return new ServiceMethod<>(this);
       }
```
## <font color =red>三、OkHttpCall.java</font>
* 网络请求实际开始执行类
### 1、继承关系
`OkHttpCall<T> implements Retrofit2.Call<T>`
### 2、主要成员变量：
`okHttp3.Call rawCall;`
### 3、伪代码执行流程
```java
       public void enqueue(Callback<T> callback){
           call = rawCall = createRawCall(){
                   Request request = serviceMethod.toRequest(args){//serviceMethod的方法
                         RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
                             contentType, hasBody, isFormEncoded, isMultipart);
                         ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
                         for (int p = 0; p < argumentCount; p++) {
                           handlers[p].apply(requestBuilder, args[p]);
                         }
                         return requestBuilder.build();
                   };
                   okhttp3.Call call = serviceMethod.callFactory.newCall(request){
                        return new okhttp3.RealCall(this, request);//实际执行网络请求类，okhttp3.Call的子类
                   };
                   return call;
               };
           call.enqueue(new okhttp3.Callback(){
              Response<T> response = parseResponse(rawResponse){
                     return responseConverter.convert(body);
              }
           }
       }
```

## <font color=red>四、整个Retrofit2封装层面主要流程涉及以上三个类，主要流程：</font>

### 步骤1、
* Retrofit2 build成员变量Call,CallAdapter.Factory,Converter.Factory,Executor

### 步骤2、
* create动态代理Proxy类，把service接口所有方法进行代理调用

### 步骤3、
* 动态代理方法中loadService构造方法解析实体类ServiceMethod，通过反射类型，注解的方式解析出方法中提供的所有参数，其中涉及到ParameterHandler类与okhttp3.Request类的参数传递关系

### 步骤4、
* 通过serviceMethod构造出OkHttpCall类，其中调用enqueue方法实现Request请求参数的填充处理，以及实际执行请求发起类RealCall的构造过程

### 步骤5、
* RealCall的执行回调中通过responseConverter对ResponseBody进行解析，返回解析需要的类型

## <font color=red>五、反射涉及类型关系：</font>
### 1、Type:
      Type type=Field.getGenericType();
      实际类型：List<T> list
      输出类型：java.util.List<T>
### 2、ParameterizedType
      获取泛型实际类型
      2.1、继承关系
         ParameterizedType extends Type
         Type type = Field.getGenericType();
         ParameterizedType pType = (ParameterizedType)type;
      2.2、getActualTypeArguments()
         Type[] actualTypes[] = pType.getActualTypeArguments();
         输出变量：actualTypes[0]
         实际类型：Set<String> set
         输出类型：class java.lang.String
      2.3、getRawType()
         Type rawType= pType.getRawType();
         输出变量：rawType
         实际类型：Set<String> set
         输出类型：interface java.util.Set
### 3、GenericArrayType
      3.1、继承关系
         GenericArrayType extends Type
      3.2、getGenericComponentType()
         GenericArrayType genericArrayType = (GenericArrayType)type;
         Type componentType = genericArrayType.getGenericComponentType();
         输出变量：componentType
         实际类型：List<String>[]
         输出类型：java.util.List<java.lang.String>
### 4、TypeVariable
      4.1、继承关系：
           TypeVariable extends Type
      4.2、getBounds()
        TypeVariable typeVariable = (TypeVariable) filed.getGenericType();
        Type[] bounds = typeVariable.getBounds();
        输出变量：bounds[0]
        实际类型：T t (注意不可为List<T>,否则会转换异常,T extends Number)
        输出类型：class java.lang.Number
      4.3、getGenericDeclaration()
        GenericDeclaration genericDeclaration = typeVariable.getGenericDeclaration();
        输出变量：genericDeclaration
        实际类型：T t
        输出类型：class com.example.retrofit.ReflectTest()（t 作为成员变量所在的实体类）
      4.4、getName()
        String name = typeVariable.getName();
        输出变量：name
        实际类型：T t
        输出类型：T
### 5、WildcardType
      5.1、继承关系
        WildcardType extends Type
      5.2、？通配符的实际类型
        Field listNum = ReflectTest.class.getDeclaredField("listNum");
        Type typeListNum = listNum.getGenericType();
        ParameterizedType parameterizedTypeListNum = (ParameterizedType) typeListNum;
        Type[] listNumActualTypeArguments = parameterizedTypeListNum.getActualTypeArguments();

        输出变量：listNumActualTypeArguments[0].getClass()
        实际类型：List<? extends Number> listNum
        输出类型：class sun.reflect.generics.reflectiveObjects.WildcardTypeImpl

      5.3、getUpperBounds()
        WildcardType wildcardTypeListNum = (WildcardType) listNumActualTypeArguments[0];
        Type[] upperBounds = wildcardTypeListNum.getUpperBounds();
        输出变量：upperBounds[0]
        实际类型：List<? extends Number> listNum
        输出类型：java.lang.Number

      5.4、getLowerBounds()
        输出变量：lowerBounds[0]
        实际类型：List<? super Number> listNum
        输出类型：java.lang.Number
