# [Lifecycle Events :: Spring Data MongoDB](https://docs.spring.io/spring-data/mongodb/reference/mongodb/lifecycle-events.html)


> 使用场景: 项目中使用template 进行 mongo操作时, 在实际执行语句前后对执行插入的数据或者查询返回的数据进行修改
> 1. <strong> 转换某个字段类型 / 存入创建时间，修改时间  </strong>
> 2. <strong> 执行后查询返回的 对象进行值的修改  </strong>

## 使用示范

```java
@Component  
public class TestListener extends AbstractMongoEventListener<Object> {  
  
    @Override  
    public void onBeforeConvert(BeforeConvertEvent<Object> event) {  
		// `insert`、`insertList` 和 `save` 操作中，在对象被 `MongoConverter` 转换为 `Document` 之前被调用。
		//塞插入时间，之类的最后操作使用
        super.onBeforeConvert(event);  
    }  
  
    @Override  
    public void onBeforeSave(BeforeSaveEvent<Object> event) {  
        super.onBeforeSave(event);  
    }  
  
    @Override  
    public void onAfterSave(AfterSaveEvent<Object> event) {  
        super.onAfterSave(event);  
    }  
  
    @Override  
    public void onAfterLoad(AfterLoadEvent<Object> event) {  
        // 在 `Document` 被从数据库中检索出来后被调用
        super.onAfterLoad(event);  
    }  
  
    @Override  
    public void onAfterConvert(AfterConvertEvent<Object> event) {
    //  在 `Document` 从数据库中被检索出来并被转换为POJO后被调用。
    // 塞属性，或者转换值时使用，防止后续再迭代进行转换
        super.onAfterConvert(event);  
    }  
  
    @Override  
    public void onAfterDelete(AfterDeleteEvent<Object> event) {  
        super.onAfterDelete(event);  
    }  
  
    @Override  
    public void onBeforeDelete(BeforeDeleteEvent<Object> event) { 
        super.onBeforeDelete(event);  
    }  
}
```

插入更新使用
`onBeforeConvert` -> `onBeforeSave` -> save ->   `onAfterSave`

查询使用

 find ->  `onAfterLoad` -> `onAfterConvert` 



