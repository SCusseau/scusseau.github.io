---
layout: post
title: Null C# isn't supported by SQL
---


> Summary 
> Using `{nameof(ClassName.Prop)}` enhances code safety, maintainability, and clarity.


Example 
```SQL
SELECT 
	d.code AS {nameof(RequestExchangeRatesDto.CurrencyCode)},
	d.name AS {nameof(RequestExchangeRatesDto.CurrencyName)},
	@year  AS  {nameof(RequestExchangeRatesDto.Year)},
	y.p1   AS {nameof(RequestExchangeRatesDto.P1)},
	y.p2   AS {nameof(RequestExchangeRatesDto.P2)}
FROM ecapex_default_exchange_rates d
LEFT JOIN ecapex_exchange_rates y ON d.code = y.code AND y.year = @year
```

With this call : 
```c#
await connection.QueryAsync<RequestExchangeRatesDto>(query, new { Year = year })
```

Using `nameof` provides several benefits:

1. **Compile-Time Safety**: The compiler verifies that the referenced property exists, catching errors if names change.
2. **Avoiding Magic Strings**: It eliminates hardcoded strings, reducing the risk of typos and silent bugs.
3. **Refactoring Support**: Property renames update automatically, unlike string literals, ensuring consistency.
4. **Readability**: It clarifies that the string represents a property name, making the code more understandable.


→ Conclusion: `nameof` enhances code safety, maintainability, and clarity.