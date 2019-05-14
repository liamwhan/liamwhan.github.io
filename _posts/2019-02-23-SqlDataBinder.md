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

That's why I wrote [`SqlDataBinder` a simple little model binder for `SqlDataReader`s class.](#class-definition)

<br/>
<br/>

----------------
<br/>
<br/>

### Usage
We'll get to the class definition in a moment, for now let's take a quick look at how to use it.

The simplest example looks like this:
```cs
var myModel = binder.Bind<MyModel>(reader);
```
In this above example `reader` is an instance of `SqlDataReader` from the `System.Data.SqlClient` namespace.


A more complete and realistic example is below:
```cs
// Initialize our connection to the DB
string connectionString = ... // your Sql connection string
string query = "SELECT * FROM SomeTable";

var binder = new SqlDataBinder(); // Initialize the Binder

using (var connection = new SqlConnection(connectionString))
{
    var command = new SqlCommand(sql, connection);

    await connection.OpenAsync();

    using (var reader = await command.ExecuteReaderAsync())
    {
        while (await reader.ReadAsync())
        {
            var myModel = binder.Bind<MyModel>(reader); // This is all you need the rest is boilerplate for connecting to SQL
        }
    }
}
```
<br/>
<br/>

----------------
<br/>
<br/>

### Attributes
The `SqlDataBinder` supports some of the most-used attribute decorations for data seralization, and some custom attributes including:
- Newtonsoft's `JsonPropertyAttribute` from the JSON.Net library
- `System.Runtime.Serialization`'s `DataMemberAttribute`
- `SqlBindIgnoreAttribute` - A custom attribute (defined below) to instruct the Binder to ignore certain properties
- `SqlBindColumn` - A custom attribute (defined below) to explicitly define the DB column name that should be bound to the property.

#### `SqlBindIgnoreAttribute` Definition
This attributes purpose is very simple: if you have properties on your Model that do not exist in the database, then you want a way to tell the binder to skip these properties. This is just a marker attribute, so we just check for its presence on the property and skip binding to that property if it is found. 

```cs
[AttributeUsage(AttributeTargets.Property)]
public class SqlBindIgnoreAttribute : Attribute
{
}
```

#### `SqlBindIgnoreAttribute` Usage
Example usage, this will case the `Id` property to be ignored by  `SqlDataBinder` but still serialized by JSON.Net.

```cs
public class MyModel 
{
    [SqlBindIgnore]
    [JsonProperty("id")]
    public string Id {get; set;}
}
```

#### `SqlBindColumnAttribute` Definition
When `SqlDataBinder` is processing a row, it needs to decide how to associate DB column to your model's properties. Without any hints, it will just try to find a column in the DB data that has the same name as the property of your Model. But alot of the time you want more control than that. 

The optional `SqlBindColumn` attribute works in much the same way as the `DataMember` and `JsonProperty` attributes do with for serialization. It specifies the name of DB column that maps to this property. You can also use `[DataMember(Name = "myColumnName")]` or `[JsonProperty("myColumnName")]` to achieve this and the result will be the same. 

```cs
[AttributeUsage(AttributeTargets.Property)]
public class SqlBindColumn : Attribute
{
    public string Name { get; set; }

    public SqlBindColumn(string name)
    {
        Name = name;
    }
}

```

#### `SqlBindColumnAttribute` Usage
Example usage, when you need to bind to property from the DB but want to rename the property when serializing it to JSON, use `SqlBindColumn`.

```cs
public class MyModel
{
    [JsonProperty("productId")]
    [SqlBindColumn("id")]
    public string Id {get; set;}
}
```
<br/>
<br/>

----------------
<br/>
<br/>

### Class Definition
Lets take a look at the `SqlDataBinder` class definition:
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

            string colName = propInfo.Name;

            // First get any JsonProperty Attributes that decorate the property to determine the DB column name. 
            // This will be overwritten by the SqlDataColumn attribute if both are present
            if (propInfo.GetCustomAttribute(typeof(JsonPropertyAttribute), true) is JsonPropertyAttribute jsonAttribute)
            {
                colName = jsonAttribute.PropertyName;
            }
            
            // Get any DataMember attributes that decorate the property to determine the DB column name
            if (propInfo.GetCustomAttribute(typeof(DataMemberAttribute), true) is DataMemberAttribute dmAttribute)
            {
                colName = dmAttribute.Name;
            }

            // Lastly overwrite the column name using the custom SqlBindColumn Attribute if present
            if (propInfo.GetCustomAttribute(typeof(SqlBindmnAttribute), true) is SqlDataColumnAttribute sqlAttribute)
            {
                colName = sqlAttribute.Name;
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

    // Base check for nullable types 
    private bool IsNullable(Type t)
    {
        return Nullable.GetUnderlyingType(t) != null;
    }

    // Gets the value type, unwraping Nullable<T> constructs
    private Type GetValueType(Type t)
    {
        if (t == typeof(string)) return t;
        if (!IsNullable(t)) return t;

        return Nullable.GetUnderlyingType(t);
    }
}
```

### Closing Remarks

You may have noticed the custom exception I throw in there `SqlDataBindingException`. I have not definied it here as it is just a straight derivation from the `Exception` class and could easily be replaced with a more generic exception like `InvalidOperationException` etc. But when I'm programming in C# (I turn exceptions off in C++) I tend to use `try/catch` blocks with multiple `catch`es so I can return a meaningful error to the User.

I hope this is useful, let me know if you used it in your project or if you have any suggestions, sound off in the comments. 
