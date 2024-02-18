---
title: 怎么实现 mybatis 自动设置创建时间更新时间
date: 2024-02-18 12:41:41
categories:
- code 
tags:
- code
- mybatis
- mybatis-plus
- 自动填充
---
## 怎么实现 mybatis 自动设置创建时间更新时间

#### :mag: 相对流行方案弊端

mybatis 提供 Interceptor 接口以插件方式提供扩展能力。互联网上大都是对数据表映射类对象中关于时间属性设置当前时间的解决方案。但这种方法无法解决 mapper.xml 写更新 SQL 或 @XXXProvider 拼 SQL 的方式插入或更新数据表。但是依托于数据表映射类本身没有问题，因为需要知道创建时间和更新时间对应的数据库字段信息，这是光拦截到 SQL 而无法判断时间相关的字段是否存在并赋值。

#### :mortar_board: 更好的选择

如果你项目中使用了 mybatis-plus 组件，恭喜你做这个决定你足够明智。 mybatis-plus 提供 MetaObjectHandler 抽象类实现公共字段自动写入能力。其大体思路是针对 @XXXProvider 拼 SQL 时将实体中标记需要自动填充的字段拼入 SQL 中，通过 metaObjectHandler 对实体属性字段填充相应值，最后带有自动填充字段的 PrepareStatement SQL 插入/更新数据表数据。

但，项目上使用自写 `BaseMapper<E,ID>` 接口和 @XXXProvider 注解实现 BaseMapperSqlSourceBuilder 类完成 SQL 拼接。但未提供对公用字段自动写入能力。

#### :mushroom: 在现状上解决问题

Interceptor 拦截的位置是执行 SQL 之前，也就是 `@Signature(type = Executor.class, method="update", args={MappedStatement.class, Object.class})` ，在 SQL 里拼接时间的字段和字段值。字段值可以直接设置的 `now()` 数据库函数，缺点是强依赖数据库。这个缺点需要通过 driver 信息找确切的数据库类型，切换时间函数。时间字段信息则是通过 `BaseMapper<E,ID>` 获取泛型 E 指向的 Class，通过属性名匹配（没办法老代码只能匹配属性名）或注解匹配找到时间字段。SQL 里拼接时间字段是通过包装 SqlSource 通过 `SqlSource#getBoundSql` 替换最终 SQL 和当前时间函数。

- `mappedStatement#getId()`，id 的值对应类全路径，从这个类全路径获取类信息并确定 `BaseMapper<E,ID>` E 指向的泛型

```java
/**
 * 表的创建时间和更新时间会随着表的更新或插入行为进行赋值. 因为需要确定表中是否有创建时间或更新时间且确定时间字段名，所以需要使用的地方
 * 的 mapper 继承 {@link BaseMapper}<br>
 * 思路，从 BaseMapper 的泛型 T 获取实体类，从实体类里面解析出创建时间和更新时间字段对应的数据库字段名，这里创建时间和更新时间是通过名称
 * 匹配的，大小写不论包含匹配.针对插入行为会增加创建时间和更新时间，针对更新行为会更新更新时间.<br>
 * <lu>
 * <li>创建时间，dCjsj、cjsj、dtCjsj、dCjrq、cjrq、dtCjrq、createTime、dCreateTime、dtCreateTime</li>
 * <li>更新时间，dGxsj, gxsj, dtGxsj, dXgsj, xgsj, dtXgsj, dZhxgsj、zhxgsj、dtZhxgsj、updateTime、dUpdateTime、dtUpdateTime</li>
 * </lu>
 *
 * @author liulili
 * @date 2024/1/25 11:24
 */
@Slf4j
@Component
@Intercepts(@Signature(type = Executor.class, method="update", args={MappedStatement.class, Object.class}))
public class AutofillCreateOrUpdateTimeInterceptor implements Interceptor {

    private final String[] CJSJ_COLUMN_NAMES = new String[] {"cjsj", "dCjsj", "dtCjsj", "cjrq", "dCjrq", "dtCjrq", "createTime", "dCreateTime", "dtCreateTime"};

    private final String[] GXSJ_COLUMN_NAMES = new String[] {"gxsj", "dGxsj", "dtGxsj", "xgsj", "dXgsj", "dtXgsj", "zhxgsj", "dZhxgsj", "dtZhxgsj", "updateTime", "dUpdateTime", "dtUpdateTime"};
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement mappedStatement = (MappedStatement) args[0];
        StatementType statement = mappedStatement.getStatementType();
        if (statement == StatementType.CALLABLE) {
            log.debug("【自动填充创建或修改时间】不支持在存储过程类型业务");
            return invocation.proceed();
        }
        SqlCommandType command = mappedStatement.getSqlCommandType();
        String id = mappedStatement.getId();
        String className = StringUtils.substring(id, 0, id.lastIndexOf("."));
        Class mapperClazz = null;
        try {
            mapperClazz = Class.forName(className);
        } catch (Throwable e) {
            if (log.isDebugEnabled()) {
                log.debug("className[{}]不是Class无法继续【自动填充创建或修改时间】的工作", className, e);
            } else if (log.isInfoEnabled()) {
                log.info("className[{}]不是Class无法继续【自动填充创建或修改时间】的工作", className);
            }
            return invocation.proceed();
        }
        Class entityClazz = findEntityClazz(mapperClazz);
        if (Objects.isNull(entityClazz)) {
            log.debug("class[{}]非接口/未继承BaseMapper接口", entityClazz);
            return invocation.proceed();
        }
        CUTimeDTO cuTimeDTO = findCreateAndUpdateTimeColumn(entityClazz);
        if (StringUtils.isBlank(cuTimeDTO.getUpdateTimeColumnName()) && StringUtils.isBlank(cuTimeDTO.getCreateTimeColumnName())) {
            log.debug("class[{}]无匹配的创建时间和更新时间字段， 请参考 AutofillCreateOrUpdateTimeInterceptor#CJSJ_COLUMN_NAMES 和 AutofillCreateOrUpdateTimeInterceptor#GXSJ_COLUMN_NAMES", entityClazz);
            return invocation.proceed();
        }
        if (command == SqlCommandType.INSERT) {
            autofillInsert(mappedStatement, args[1], entityClazz, cuTimeDTO);
        } else if (command == SqlCommandType.UPDATE) {
            autofillUpdate(mappedStatement, args[1], entityClazz, cuTimeDTO);
        }
        return invocation.proceed();
    }

    private void autofillUpdate(MappedStatement mappedStatement, Object param, Class entityClazz, CUTimeDTO cuTimeDTO) {
        String updateTimeColumn = cuTimeDTO.getUpdateTimeColumnName();
        if (StringUtils.isBlank(updateTimeColumn)) {
            return;
        }
        BoundSql boundSql = mappedStatement.getBoundSql(param);
        String sql = boundSql.getSql();
        if (StringUtils.containsIgnoreCase(sql, updateTimeColumn)) {
            autofillUTime(param, cuTimeDTO, entityClazz);
            return;
        }
        SqlSource sqlSource = mappedStatement.getSqlSource();
        SqlSource decoderSqlSource = new AutoFillUTimeUpdateSqlSource(sqlSource, updateTimeColumn, "now()");
        BeanUtil.setProperty(mappedStatement, "sqlSource", decoderSqlSource);
    }

    private void autofillInsert(MappedStatement mappedStatement, Object param, Class entityClazz, CUTimeDTO cuTimeDTO) {
        String createColumnName = cuTimeDTO.getCreateTimeColumnName();
        String updateColumnName = cuTimeDTO.getUpdateTimeColumnName();
        BoundSql boundSql = mappedStatement.getBoundSql(param);
        String sql = boundSql.getSql();
        List<String> addColumn = new ArrayList<>(2);
        if (StringUtils.isNotBlank(createColumnName) && !StringUtils.containsIgnoreCase(sql, createColumnName)) {
            addColumn.add(createColumnName);
        }
        if (StringUtils.isNotBlank(updateColumnName) && !StringUtils.containsIgnoreCase(sql, updateColumnName)) {
            addColumn.add(updateColumnName);
        }
        if (CollectionUtils.isEmpty(addColumn)) {
            autofillCUTime(param, cuTimeDTO, entityClazz);
            return;
        }
        String columnName = addColumn.stream().collect(Collectors.joining(","));
        String columnValue = addColumn.stream().map(column -> "now()").collect(Collectors.joining(","));
        SqlSource sqlSource = mappedStatement.getSqlSource();
        SqlSource decoderSqlSource = new AutoFillCUTimeInsertSqlSource(sqlSource, columnName, columnValue);
        BeanUtil.setProperty(mappedStatement, "sqlSource", decoderSqlSource);
    }

    private void autofillCUTime(Object param, CUTimeDTO cuTimeDTO, Class entityClazz) {
        if (Objects.isNull(param)) {
            return;
        }
        if (param.getClass().isAssignableFrom(entityClazz)) {
            Optional.ofNullable(cuTimeDTO.getCreateTimePropertyName())
                    .ifPresent(createColumnName -> BeanUtil.setProperty(param, createColumnName, Calendar.getInstance().getTime()));
            Optional.ofNullable(cuTimeDTO.getUpdateTimePropertyName())
                    .ifPresent(updateColumnName -> BeanUtil.setProperty(param, updateColumnName, Calendar.getInstance().getTime()));
            return;
        }
        if (param instanceof Collection) {
            Collection paramColl = (Collection) param;
            paramColl.stream().forEach(sparam -> autofillCUTime(sparam, cuTimeDTO, entityClazz));
            return;
        }
        if (param instanceof Map) {
            Map paramMap = (Map) param;
            Set<Map.Entry> entries = paramMap.entrySet();
            entries.stream().forEach(entry -> autofillCUTime(entry.getValue(), cuTimeDTO, entityClazz));
            return;
        }
        if (param.getClass().isPrimitive() || param.getClass().isEnum()) {
            return;
        }
        if (param.getClass().isArray()) {
            Object[] paramArr = (Object[]) param;
            Arrays.stream(paramArr).forEach(obj -> autofillCUTime(obj, cuTimeDTO, entityClazz));
            return;
        }
        Field[] fields = param.getClass().getDeclaredFields();
        Arrays.stream(fields).forEach(field -> {
            Object property = null;
            try {
                property = BeanUtil.getProperty(param, field.getName());
            } catch (Exception e) {
                log.debug("param[{}]属性【{}】获取属性值失败", param, field.getName(), e);
            }
            autofillCUTime(property, cuTimeDTO, entityClazz);
        });
    }

    private void autofillUTime(Object param, CUTimeDTO cuTimeDTO, Class entityClazz) {
        if (Objects.isNull(param) || param.getClass().isPrimitive() || param.getClass().isEnum()) {
            return;
        }
        if (param.getClass().isAssignableFrom(entityClazz)) {
            Optional.ofNullable(cuTimeDTO.getUpdateTimePropertyName()).ifPresent(updateColumnName -> BeanUtil.setProperty(param, updateColumnName, Calendar.getInstance().getTime()));
            return;
        }
        if (param instanceof Collection) {
            Collection paramColl = (Collection) param;
            paramColl.stream().forEach(sparam -> autofillUTime(sparam, cuTimeDTO, entityClazz));
            return;
        }
        if (param instanceof Map) {
            Map paramMap = (Map) param;
            Set<Map.Entry> entries = paramMap.entrySet();
            entries.stream().forEach(entry -> autofillUTime(entry.getValue(), cuTimeDTO, entityClazz));
            return;
        }
        if (param.getClass().isArray()) {
            Object[] paramArr = (Object[]) param;
            Arrays.stream(paramArr).forEach(obj -> autofillUTime(obj, cuTimeDTO, entityClazz));
            return;
        }
        Field[] fields = param.getClass().getDeclaredFields();
        Arrays.stream(fields).forEach(field -> {
            Object property = null;
            try {
                property = BeanUtil.getProperty(param, field.getName());
            } catch (Exception e) {
                log.debug("param[{}]属性【{}】获取属性值失败", param, field.getName(), e);
            }
            autofillUTime(property, cuTimeDTO, entityClazz);
        });
    }

    private CUTimeDTO findCreateAndUpdateTimeColumn(Class entityClazz) {
        Field[] fields = entityClazz.getDeclaredFields();
        Map<String, Field> fieldMap = Arrays.stream(fields).collect(Collectors.toMap(Field::getName, field -> field));
        Set<String> fieldKeys = fieldMap.keySet();
        CUTimeDTO CUTimeDTO = new CUTimeDTO();
        Optional<String> cjsjFieldNameOptional = fieldKeys.stream().filter(fieldKey -> Arrays.stream(CJSJ_COLUMN_NAMES)
                .anyMatch(columnName -> StringUtils.equalsIgnoreCase(columnName, fieldKey))).findFirst();
        cjsjFieldNameOptional.ifPresent(cjsjFieldName -> CUTimeDTO.setCreateTimePropertyName(cjsjFieldName));
        CUTimeDTO.setCreateTimeColumnName(getColumnNameByColumnAnno(cjsjFieldNameOptional, fieldMap));
        Optional<String> gxsjFieldNameOptional = fieldKeys.stream().filter(fieldKey -> Arrays.stream(GXSJ_COLUMN_NAMES)
                .anyMatch(columnName -> StringUtils.equalsIgnoreCase(columnName, fieldKey))).findFirst();
        gxsjFieldNameOptional.ifPresent(gxsjFieldName -> CUTimeDTO.setUpdateTimePropertyName(gxsjFieldName));
        CUTimeDTO.setUpdateTimeColumnName(getColumnNameByColumnAnno(gxsjFieldNameOptional, fieldMap));
        return CUTimeDTO;
    }

    private String getColumnNameByColumnAnno(Optional<String> fieldNameOptional, Map<String, Field> fieldMap) {
        String columnName = null;
        if (fieldNameOptional.isPresent()) {
            String fieldName = fieldNameOptional.get();
            Field field = fieldMap.get(fieldName);
            Column column = field.getAnnotation(Column.class);
            columnName = column.name();
        }
        return columnName;
    }

    private Class findEntityClazz(Class mapperClazz) {
        if (!mapperClazz.isInterface()) {
            return null;
        }
        Type[] interfaces = mapperClazz.getGenericInterfaces();
        if (Objects.isNull(interfaces)) {
            return null;
        }
        Optional<ParameterizedType> baseMapperTypeOptional = Arrays.stream(interfaces)
                .filter(iface -> iface instanceof ParameterizedType)
                .map(iface -> (ParameterizedType) iface)
                .filter(iface -> ((Class) iface.getRawType()).isAssignableFrom(BaseMapper.class))
                .findFirst();
        if (!baseMapperTypeOptional.isPresent()) {
            return null;
        }
        ParameterizedType baseMapperType = baseMapperTypeOptional.get();
        return (Class) baseMapperType.getActualTypeArguments()[0];
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }

}
```

设计的 DTO 用于确定属性对应的创建时间字段属性和更新时间字段属性。

```java
@Getter
@Setter
@Accessors(chain = true)
@NoArgsConstructor
public class CUTimeDTO {

    private String createTimePropertyName;

    private String updateTimePropertyName;

    private String createTimeColumnName;

    private String updateTimeColumnName;
}
```

包装对应的 SqlSource 在获取最后的 SQL （`SqlSource#getBoundSql`）中拼接创建和更新时间脚本。不在具体的 SqlSource 里面完成字段拼接加上预处理字段，是因为 mybatis 支持多种 SqlSource 包含 `StaticSqlSource`、`ProviderSqlSource`、`RawSqlSource`、`DynamicSqlSource`，且他们可以组合出现，可见还是有一定的复杂度的。所以才选择用包装类完成字段填充。这种是不建议自动填充那种包含不同值的字段的，因为这样会让预处理 SQL 没有发挥作用。

```java
@AllArgsConstructor
public class AutoFillUTimeUpdateSqlSource implements SqlSource {

    private SqlSource sqlSource;

    private String columnName;

    private String columnValue;


    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        BoundSql boundSql = this.sqlSource.getBoundSql(parameterObject);
        replaceBoundSql(boundSql);
        return boundSql;
    }
    private void replaceBoundSql(BoundSql boundSql) {
        String sql = boundSql.getSql();
        String newSql = StringUtils.replaceIgnoreCase(sql, "set ", "set " + columnName + "=" + columnValue + ",");
        BeanUtil.setProperty(boundSql, "sql", newSql);
    }
}
```

```java
@AllArgsConstructor
public class AutoFillCUTimeInsertSqlSource implements SqlSource  {

    private SqlSource sqlSource;

    private String columnName;

    private String columnValue;

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        BoundSql boundSql = this.sqlSource.getBoundSql(parameterObject);
        replaceBoundSql(boundSql);
        return boundSql;
    }

    private void replaceBoundSql(BoundSql boundSql) {
        String sql = boundSql.getSql();
        Pattern pattern = Pattern.compile("\\(");
        Matcher matcher = pattern.matcher(sql);
        String newSql = sql;
        if (matcher.find()) {
            int index = matcher.start();
            newSql = sql.substring(0, index + 1) + columnName + "," + sql.substring(index + 1);
        }

        int index = StringUtils.indexOfIgnoreCase(newSql, "values");
        int index1 = index + "values".length();
        while(index1 < newSql.length() && index1 > 0) {
            index1 = index1 + 1;
            char next = newSql.charAt(index1);
            if (next == ' ' || next == '\\' || next == 'n') {
                continue;
            }
            if (next == '(') {
                break;
            }
            index1 = StringUtils.indexOfIgnoreCase(newSql, "values", index1);
        }
        if (index1 == -1) {
            return;
        }
        String replace = StringUtils.substring(newSql, index, index1 + 1);
        newSql = newSql.replace(replace, replace + columnValue + ",");
        BeanUtil.setProperty(boundSql, "sql", newSql);
    }
}
```

#### :question: 猜测你会有这样的疑问

为什么不让项目直接集成 mybatis-plus 修改 pojo 就能快速解决问题，不用这么复杂。当然我统一这个思路，但这个思路适合于 pojo 少，且使用 `@Table` 、`@Column` 等数据库型的注解的项目。否则，在大项目中还是工作量及风险还是比较高。但这不影响我推荐使用 mybatis-plus。
