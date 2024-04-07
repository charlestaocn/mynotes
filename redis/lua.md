
```java
   public <T> T evalLua(RedisScript<T> script, List keys, Object... args) {
        return (T) redisTemplate.execute(script, new FastJson2JsonRedisSerializer<>(), new FastJson2JsonRedisSerializer<>(), keys, args);
    }

    public <T> T evalLua(String script, Class<T> resultTypeClz, List keys, Object... args) {
        DefaultRedisScript<T> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(script);
        redisScript.setResultType(resultTypeClz);
        return evalLua(redisScript, keys, args);
    }

    public <T> T evalLua(ScriptSource scriptSource, Class<T> resultTypeClz, List keys, Object... args) {
        DefaultRedisScript<T> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(scriptSource);
        redisScript.setResultType(resultTypeClz);
        return evalLua(redisScript, keys, args);
    }

```


```java


/**
 * Redis使用FastJson序列化
 *
 * @author ruoyi
 */
public class FastJson2JsonRedisSerializer<T> implements RedisSerializer<T> {
    public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

    static final Filter AUTO_TYPE_FILTER = JSONReader.autoTypeFilter(Constants.JSON_WHITELIST_STR);

    private Class<T> clazz;

    public FastJson2JsonRedisSerializer() {
        super();
    }

    public FastJson2JsonRedisSerializer(Class<T> clazz) {
        super();
        this.clazz = clazz;
    }

    @Override
    public byte[] serialize(T t) throws SerializationException {
        if (t == null) {
            return new byte[0];
        }
        return JSON.toJSONString(t, JSONWriter.Feature.WriteClassName).getBytes(DEFAULT_CHARSET);
    }

    @Override
    public T deserialize(byte[] bytes) throws SerializationException {
        if (bytes == null || bytes.length <= 0) {
            return null;
        }

        String str = new String(bytes, DEFAULT_CHARSET);

        if (null != clazz) {
            return JSON.parseObject(str, clazz, AUTO_TYPE_FILTER);
        }
        return JSON.parseObject(str, new TypeReference<T>() {
        }, AUTO_TYPE_FILTER);
    }
}

```