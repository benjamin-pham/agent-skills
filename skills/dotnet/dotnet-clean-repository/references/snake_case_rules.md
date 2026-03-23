# Snake_case Naming Convention Reference

## Conversion Rules: PascalCase → snake_case

### Basic Rule
Insert an underscore before each uppercase letter (except the first), then lowercase everything.

```
FirstName     → first_name
OrderItem     → order_item
CreatedAt     → created_at
IsDeleted     → is_deleted
CategoryId    → category_id
```

### Consecutive Uppercase (Acronyms)
Treat acronym as a single word when followed by lowercase; split between consecutive uppercase groups otherwise.

```
HTMLParser    → html_parser
IOStream     → io_stream
APIKey        → api_key
UserDTO       → user_dto
XMLHttpRequest → xml_http_request
```

### Table Name (Plural snake_case)

| Singular Class | Plural Table |
|---|---|
| `Product` | `products` |
| `Category` | `categories` |
| `OrderItem` | `order_items` |
| `ProductCategory` | `product_categories` |
| `Address` | `addresses` |
| `Company` | `companies` |
| `Status` | `statuses` |
| `Person` | `people` |
| `Child` | `children` |

### Pluralization Rules
1. Most nouns: add `s` → `products`
2. Ends in `s`, `sh`, `ch`, `x`, `z`: add `es` → `addresses`, `statuses`
3. Ends in consonant + `y`: change `y` to `ies` → `categories`, `companies`
4. Ends in `f`/`fe`: change to `ves` → `wolves`, `lives`
5. Irregular: memorize → `person/people`, `child/children`, `mouse/mice`

### Database Object Naming

| Object | Pattern | Example |
|---|---|---|
| Table | `{snake_case_plural}` | `order_items` |
| Column | `{snake_case}` | `first_name` |
| Primary Key | `id` | `id` |
| Foreign Key Column | `{referenced_table_singular}_id` | `category_id` |
| FK Constraint | `fk_{table}_{column}` | `fk_products_category_id` |
| Index | `ix_{table}_{columns}` | `ix_products_name` |
| Unique Index | `uq_{table}_{columns}` | `uq_users_email` |
| Composite Index | `ix_{table}_{col1}_{col2}` | `ix_order_items_order_id_product_id` |
