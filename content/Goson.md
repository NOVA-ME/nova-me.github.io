+++
title = "Goson使用总结"
date = 2019-10-03

[taxonomies]
tags = ["Java"]
categories = ["Json"]
+++
这篇文章是对Gson的内部实现原理进行一个总结
<!-- more -->
## 创建方法
Gson可以由两种方式创建，一种是通过Gson.Builder()的方式，另一种则是通过new Gson()来创建。

那这两种创建方式有什么不同呢？

第一种Gson.Builder()的方法没有使用反射，而new Gson()是使用了反射的。

在Gson内部存在着TypeAdapter，也就是根据不同的类型来适配，当我们使用Gson.Builder()创建时，使用的是自定义的TypeAdapter而不是默认的TypeAdapter，因此也就出现了一个为反射，一个不是反射

```java
Gson(Excluder excluder, FieldNamingStrategy fieldNamingPolicy, Map<Type, InstanceCreator<?>> instanceCreators, boolean serializeNulls, boolean complexMapKeySerialization, boolean generateNonExecutableGson, boolean htmlSafe, boolean prettyPrinting, boolean serializeSpecialFloatingPointValues, LongSerializationPolicy longSerializationPolicy, List<TypeAdapterFactory> typeAdapterFactories) {
        this.calls = new ThreadLocal();
        this.typeTokenCache = Collections.synchronizedMap(new HashMap());
        this.deserializationContext = new JsonDeserializationContext() {
            public <T> T deserialize(JsonElement json, Type typeOfT) throws JsonParseException {
                return Gson.this.fromJson(json, typeOfT);
            }
        };
        this.serializationContext = new JsonSerializationContext() {
            public JsonElement serialize(Object src) {
                return Gson.this.toJsonTree(src);
            }

            public JsonElement serialize(Object src, Type typeOfSrc) {
                return Gson.this.toJsonTree(src, typeOfSrc);
            }
        };
        this.constructorConstructor = new ConstructorConstructor(instanceCreators);
        this.serializeNulls = serializeNulls;
        this.generateNonExecutableJson = generateNonExecutableGson;
        this.htmlSafe = htmlSafe;
        this.prettyPrinting = prettyPrinting;
        ArrayList factories = new ArrayList();
        factories.add(TypeAdapters.JSON_ELEMENT_FACTORY);
        factories.add(ObjectTypeAdapter.FACTORY);
        factories.add(excluder);
        factories.addAll(typeAdapterFactories);
        factories.add(TypeAdapters.STRING_FACTORY);
        factories.add(TypeAdapters.INTEGER_FACTORY);
        factories.add(TypeAdapters.BOOLEAN_FACTORY);
        factories.add(TypeAdapters.BYTE_FACTORY);
        factories.add(TypeAdapters.SHORT_FACTORY);
        factories.add(TypeAdapters.newFactory(Long.TYPE, Long.class, this.longAdapter(longSerializationPolicy)));
        factories.add(TypeAdapters.newFactory(Double.TYPE, Double.class, this.doubleAdapter(serializeSpecialFloatingPointValues)));
        factories.add(TypeAdapters.newFactory(Float.TYPE, Float.class, this.floatAdapter(serializeSpecialFloatingPointValues)));
        factories.add(TypeAdapters.NUMBER_FACTORY);
        factories.add(TypeAdapters.CHARACTER_FACTORY);
        factories.add(TypeAdapters.STRING_BUILDER_FACTORY);
        factories.add(TypeAdapters.STRING_BUFFER_FACTORY);
        factories.add(TypeAdapters.newFactory(BigDecimal.class, TypeAdapters.BIG_DECIMAL));
        factories.add(TypeAdapters.newFactory(BigInteger.class, TypeAdapters.BIG_INTEGER));
        factories.add(TypeAdapters.URL_FACTORY);
        factories.add(TypeAdapters.URI_FACTORY);
        factories.add(TypeAdapters.UUID_FACTORY);
        factories.add(TypeAdapters.LOCALE_FACTORY);
        factories.add(TypeAdapters.INET_ADDRESS_FACTORY);
        factories.add(TypeAdapters.BIT_SET_FACTORY);
        factories.add(DateTypeAdapter.FACTORY);
        factories.add(TypeAdapters.CALENDAR_FACTORY);
        factories.add(TimeTypeAdapter.FACTORY);
        factories.add(SqlDateTypeAdapter.FACTORY);
        factories.add(TypeAdapters.TIMESTAMP_FACTORY);
        factories.add(ArrayTypeAdapter.FACTORY);
        factories.add(TypeAdapters.ENUM_FACTORY);
        factories.add(TypeAdapters.CLASS_FACTORY);
        factories.add(new CollectionTypeAdapterFactory(this.constructorConstructor));
        factories.add(new MapTypeAdapterFactory(this.constructorConstructor, complexMapKeySerialization));
        factories.add(new ReflectiveTypeAdapterFactory(this.constructorConstructor, fieldNamingPolicy, excluder));
        this.factories = Collections.unmodifiableList(factories);
    }
```
那Gson如何判断我们使用的是哪一种方法创建呢？其实就是通过factories这个对象 add Factory的顺序来控制的。每次获取Adapter的时候都需要遍历Factories，而我们自定义的TypeAdapter在add时是会默认添加到靠前的位置。所以这样就避免了Gson的反射解析

## 如何实现解析
Gson中存在一个JsonParser类，它的作用就是将json串解析成JsonElement对象。

```java
public JsonElement parse(String json) throws JsonSyntaxException {
        return this.parse((Reader)(new StringReader(json)));
    }

    public JsonElement parse(Reader json) throws JsonIOException, JsonSyntaxException {
        try {
            JsonReader e = new JsonReader(json);
            JsonElement element = this.parse(e);
            if(!element.isJsonNull() && e.peek() != JsonToken.END_DOCUMENT) {
                throw new JsonSyntaxException("Did not consume the entire document.");
            } else {
                return element;
            }
        } catch (MalformedJsonException var4) {
            throw new JsonSyntaxException(var4);
        } catch (IOException var5) {
            throw new JsonIOException(var5);
        } catch (NumberFormatException var6) {
            throw new JsonSyntaxException(var6);
        }
    }

    public JsonElement parse(JsonReader json) throws JsonIOException, JsonSyntaxException {
        boolean lenient = json.isLenient();
        json.setLenient(true);

        JsonElement e;
        try {
            e = Streams.parse(json);
        } catch (StackOverflowError var8) {
            throw new JsonParseException("Failed parsing JSON source: " + json + " to Json", var8);
        } catch (OutOfMemoryError var9) {
            throw new JsonParseException("Failed parsing JSON source: " + json + " to Json", var9);
        } finally {
            json.setLenient(lenient);
        }

        return e;
    }
```

注意上面的第三个方法，里面实际上返回的是Stream.parse(json)

```java
public static JsonElement parse(JsonReader reader) throws JsonParseException {
        boolean isEmpty = true;

        try {
            reader.peek();
            isEmpty = false;
            return (JsonElement)TypeAdapters.JSON_ELEMENT.read(reader);
        } catch (EOFException var3) {
            if(isEmpty) {
                return JsonNull.INSTANCE;
            } else {
                throw new JsonSyntaxException(var3);
            }
        } catch (MalformedJsonException var4) {
            throw new JsonSyntaxException(var4);
        } catch (IOException var5) {
            throw new JsonIOException(var5);
        } catch (NumberFormatException var6) {
            throw new JsonSyntaxException(var6);
        }
    }
```

这里又返回了TypeAdapters.JSON_ELEMENT.read(reader)方法，那我们就来看看read方法又返回了什么。

```java
public JsonElement read(JsonReader in) throws IOException {
                switch(TypeAdapters.SyntheticClass_1.$SwitchMap$com$google$gson$stream$JsonToken[in.peek().ordinal()]) {
                case 1:
                    String number = in.nextString();
                    return new JsonPrimitive(new LazilyParsedNumber(number));
                case 2:
                    return new JsonPrimitive(Boolean.valueOf(in.nextBoolean()));
                case 3:
                    return new JsonPrimitive(in.nextString());
                case 4:
                    in.nextNull();
                    return JsonNull.INSTANCE;
                case 5:
                    JsonArray array = new JsonArray();
                    in.beginArray();

                    while(in.hasNext()) {
                        array.add(this.read(in));
                    }

                    in.endArray();
                    return array;
                case 6:
                    JsonObject object = new JsonObject();
                    in.beginObject();

                    while(in.hasNext()) {
                        object.add(in.nextName(), this.read(in));
                    }

                    in.endObject();
                    return object;
                case 7:
                case 8:
                case 9:
                case 10:
                default:
                    throw new IllegalArgumentException();
                }
            }
```

从这上面的case我们不难看出，这里实际上就是根据不同的数据类型进行不同的数据解析。例如case 6,针对的是一个object类型，也就获取它的name和value放入object中，case 5则是针对的Array类型。

所以实际上就是解析Json，然后返回一个调用者想要的对象

## 一句话总结

两种创建方式，区别在于一个是反射一个不是反射。如果想要避免使用反射，自定义TypeAdapter即可，factories会自动将其添加到list的前面，当使用时就会优先遍历到指定的adapter。
JsonParser对Json串进行解析处理，根据对应的数据类型最终返回给用户一个指定的对象。这也就是我们创建一个对象实体类的原因