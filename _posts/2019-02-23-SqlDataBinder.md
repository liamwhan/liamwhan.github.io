---

layout: post
categories: csharp sql
title: A simple Model Binder for SqlDataReader in C#

---

If you've ever used `System.Data.SqlClient` in a .NET Core targetted project, you've probably been seen or written data transformation code that looks like this:

```cs
while (await reader.ReadAsync())
{
    var model = new MyModel
    {
        Date = reader.GetDateTime(1),
        Duration = reader.IsDBNull(2) ? null : reader.GetTimeSpan(2).ToString(),
        IsExisting = reader.GetBoolean(3),
        Name = reader.IsDBNull(3) ? null : reader.GetString(4)
    };
}
```

If all of your data is as concise as the example above, then I guess this approach is fine, but when you're dealing with 20+ column datasets, things get ugly very quick. 

That's why I wrote a simple little model binder for the `SqlDataReader` class:

```cs
public class SqlDataBinder
{
    public T Bind<T>(SqlDataReader reader)
        where T : class, new()
    {
        var type = typeof(T);
        var properties = type.GetProperties();
        var instance = new T();

        foreach (var propInfo in properties)
        {
            if (propInfo.GetCustomAttribute(typeof(SqlBindIgnoreAttribute)) is SqlBindIgnoreAttribute ignoreAttribute) continue; // Skip Properties with this Attribute

            string colName = string.Empty;

            if (propInfo.GetCustomAttribute(typeof(DataMemberAttribute), true) is DataMemberAttribute dmAttribute)
            {
                colName = dmAttribute.Name;
            }

            // Prioritise JsonProperty Attribute if present
            if (propInfo.GetCustomAttribute(typeof(JsonPropertyAttribute), true) is JsonPropertyAttribute jsonAttribute)
            {
                colName = jsonAttribute.PropertyName;
            }

            if (string.IsNullOrEmpty(colName)) continue;

            var propName = propInfo.Name;
            var propType = propInfo.PropertyType;
            var propTypeName = propType.Name;
            int colIndex;
            string sqlTypeName;
            try
            {
                colIndex = reader.GetOrdinal(colName);
                sqlTypeName = reader.GetDataTypeName(colIndex);
            }
            catch
            {
                // column not found in Sql Data, skip
                continue;
            }

            // DBNull values
            if (reader.IsDBNull(colIndex))
            {
                if (!IsNullable(propType) && propType != typeof(string))
                {
                    throw new SqlDataBindingException($"SqlDataReader column '{colName}' contains a DB Null, but property '{propName}' is a non-nullable type: {propTypeName}");
                }

                propInfo.SetValue(instance, null);
                continue;
            }

            // Handle Enums
            try
            {
                if (propType.IsEnum)
                {
                    if (sqlTypeName == "int" || sqlTypeName == "varchar")
                    {
                        int value = Cast<int>(reader[colName]);
                        propInfo.SetValue(instance, Convert.ChangeType(Enum.Parse(propType, value.ToString()), propType));
                        continue;
                    }
                }

            }
            catch
            {
                throw new SqlDataBindingException($"Property '{propName}' has Enum type: '{propTypeName}', but Sql data for column '{colName}' could not be parsed as a '{propTypeName}");
            }


            // Primitive types
            try
            {
                propInfo.SetValue(instance, Convert.ChangeType(reader[colName], GetValueType(propType)));
            }
            catch
            {
                throw new SqlDataBindingException($"Error setting property '{propName}' type conversion from SqlDataType '{sqlTypeName}' to runtime type '{propTypeName}'");
            }
        }

        return instance;
    }



    private T Cast<T>(object input)
    {
        return (T)Convert.ChangeType(input, typeof(T));
    }

    private bool IsNullable(Type t)
    {
        return Nullable.GetUnderlyingType(t) != null;
    }

    private Type GetValueType(Type t)
    {
        if (t == typeof(string)) return t;
        if (!IsNullable(t)) return t;

        return Nullable.GetUnderlyingType(t);
    }


}

```

