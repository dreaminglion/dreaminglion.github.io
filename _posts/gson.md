---
layout: post
title:  "google gson"
categories: gson
---

google gson 的使用，业务场景：接口定义了默认值为 true 的 boolean 字段，如果服务器同学忘记给该字段，使用 gson 默认的解析为 false，会造成功能错误无法使用。

<!-- more -->

## app 端定义的 bean 对象

```
public class GsonBean {

    public int age;
    public boolean beastworker;
    public boolean like;
    public String name;
    public String friend;
}

```

## server 端返回的 gson 数据

```
{
  "age" : 24,
  "beastworker" : true,
  "name" : "lion"
}

```

## 缺省值的配置

```
ArrayMap<String, Object> map = new ArrayMap<>();
map.put("like", true);
map.put("friend", "yy");
```

## gson 反序列发解析

```
public class GsonUtils {

    private static final String TAG = "GsonUtils";

    public static String serverData = "{\"age\":24,\"beastworker\":true,\"name\":\"lion\"}";

    public static void gsonTest() {
        ArrayMap<String, Object> map = new ArrayMap<>();
        map.put("like", true);
        map.put("friend", "yy");

        Gson gson = new GsonBuilder().registerTypeAdapter(
                GsonBean.class, new BeastJsonDeserializer<GsonBean>(map)).create();
        GsonBean gsonBean = gson.fromJson(serverData, GsonBean.class);

        // print {"age":24,"beastworker":true,"friend":"yy","like":true,"name":"lion"}
        Log.i(TAG, gson.toJson(gsonBean));
    }

    /**
     * Gson 自定义服务器缺省默认值。
     */
    static class BeastJsonDeserializer<T> implements JsonDeserializer<T> {

        private ArrayMap<String, Object> defaultValues;

        public BeastJsonDeserializer(ArrayMap<String, Object> defaultValues) {
            this.defaultValues = defaultValues;
        }

        @Override
        public T deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context) throws JsonParseException {
            JsonObject jsonObject = json.getAsJsonObject();
            if (defaultValues != null) {
                for (String key : defaultValues.keySet()) {
                    if (jsonObject.get(key) == null) {
                        if (defaultValues.get(key) instanceof String) {
                            jsonObject.addProperty(key, (String) defaultValues.get(key));
                        } else if (defaultValues.get(key) instanceof Character) {
                            jsonObject.addProperty(key, (Character) defaultValues.get(key));
                        } else if (defaultValues.get(key) instanceof Number) {
                            jsonObject.addProperty(key, (Number) defaultValues.get(key));
                        } else if (defaultValues.get(key) instanceof Boolean) {
                            jsonObject.addProperty(key, (Boolean) defaultValues.get(key));
                        }
                    }
                }
            }
            return new Gson().fromJson(jsonObject.toString(), typeOfT);
        }
    }

}

```

1. 定义缺省的字段对应的默认值。
 - map.put("like", true); map.put("friend", "yy");
2. 定义反序列发解析器 BeastJsonDeserializer<T>;
 - defaultValues 中通过 key 看服务器是有 key 对应的字段。
 - jsonObject.get(key) == null 没有改字段，就取 defaultValues key 对应的 value 作为解析的默认值。
3. 使用 GsonBuilder registerTypeAdapter 注册我们的反序列化器，得到一个 gson 对象。
4. 和普通 gson 一样使用 GsonBean gsonBean = gson.fromJson(serverData, GsonBean.class);
5. 打印得到的结构 {"age":24,"beastworker":true,"friend":"yy","like":true,"name":"lion"}
6. 如果服务器配置了「like」，「friend」对应的值，缺省默认值就不会被使用。
