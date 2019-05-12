---

layout: post
categories: csharp sql
title: A simple Model Binder for SqlDataReader in C#

---


```cs
public class SqlDataBinder : ISqlDataBinder
    {

        /// <summary>
        /// <para>Binds a SQL Result to a single object instance of type <typeparam name="T">T</typeparam></para>
        /// <para>Uses reflection to inspect class properties and attributes to determine mapping from <see cref="SqlDataReader"/> columns to class properties</para>
        /// <para>I created a custom attribute to use with this <see cref="SqlBindIgnoreAttribute"/> that works in much the same way as <see cref="JsonIgnoreAttribute"/>
        /// and can be applied to properties so that they are skipped. Useful if you need something serialized into JSON but don't want the reader to try and match it.</para>
        /// <para>This was written on the fly towards the end of the Kennards API project so it probably doesn't cover every possible failure case but it handles
        /// all primitive types, nullables and Enums. Nested class properties are not supported by SQL can't return those anyway</para>
        /// </summary>
        /// <typeparam name="T">The type to bind to</typeparam>
        /// <param name="reader"><see cref="SqlDataReader"/> instance</param>
        /// <returns></returns>
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

