# RiwiMarket S.A.S. — Academic Database Project

## Project description
RiwiMarket S.A.S. is a company that distributes consumer products to supermarkets
and neighborhood stores across several Colombian cities. All business data
(suppliers, products, categories, warehouses, purchases, and inventory movements)
used to be tracked in a single shared Excel file, which caused duplicated
suppliers, duplicated products, inconsistent category names, and unreliable
purchase order numbers. This project normalizes that raw data into a relational
database, up to Third Normal Form (3NF), and implements it in MySQL with full
DDL, DML, and business-analysis queries.

## Technologies used
- MySQL 8.x
- CSV files for data loading (`LOAD DATA LOCAL INFILE`)
- Python 3 (pandas) — used only offline, to clean and deduplicate the raw
  dataset into the CSV files that are loaded into the database

## Database engine
MySQL 8.x

## Normalization process
The raw file (`raw_dataset_riwibuild_practice.csv`) was a single flat table
with 25 columns mixing suppliers, warehouses, products, purchases, and
inventory movements in the same rows. The full analysis and normalization
reasoning (1NF → 2NF → 3NF) is documented in:
- `01_analisis_calidad_datos.md` — data quality analysis (duplicated
  suppliers/warehouses/products, inconsistent categories/units, unreliable
  purchase codes, null values, functional dependencies).
- `02_normalizacion.md` — step-by-step normalization to 3NF.

Key takeaway: `unit_price` was moved out of the product entity into the
purchase entity, since price depends on the transaction (date, supplier
negotiation), not on the product itself — keeping it on the product would
have reintroduced a transitive dependency.

## Database structure
All tables use the `rm_` prefix and English attribute names, as required.

| Table | Purpose | Primary Key | Foreign Keys |
|---|---|---|---|
| `rm_cities` | Master list of cities | `city_id` | — |
| `rm_categories` | Master list of product categories | `category_id` | — |
| `rm_suppliers` | Deduplicated suppliers (unique by tax_id) | `supplier_id` | `city_id` → rm_cities |
| `rm_warehouses` | Deduplicated warehouses (unique by address) | `warehouse_id` | `city_id` → rm_cities |
| `rm_products` | Deduplicated products (unique by SKU) | `product_id` | `category_id` → rm_categories |
| `rm_employees` | Employees responsible for purchases/movements | `employee_id` | — |
| `rm_purchases` | One row per purchase line (product + supplier + warehouse + price) | `purchase_id` | `supplier_id`, `product_id`, `warehouse_id` |
| `rm_movements` | All inventory movements (IN / OUT / adjustments) | `movement_id` | `product_id`, `warehouse_id`, `employee_id`, `purchase_id` (nullable) |

Constraints applied: `PRIMARY KEY`, `FOREIGN KEY` (with `ON DELETE
RESTRICT`/`SET NULL` as appropriate), `UNIQUE` (tax_id, SKU, movement_code,
city_name, category_name, employee full_name), `NOT NULL`, and `CHECK`
(quantity > 0, unit_price > 0, valid movement_type, valid unit_measure).

## Entity Relationship Model
The ER model (entities, attributes, primary/foreign keys, relationships and
cardinalities) should be built with a diagramming tool (Draw.io, Lucidchart,
Visual Paradigm, or StarUML) reflecting the structure above, and exported as
`ER_model.pdf`. Suggested cardinalities:
- `rm_cities (1) — (N) rm_suppliers`
- `rm_cities (1) — (N) rm_warehouses`
- `rm_categories (1) — (N) rm_products`
- `rm_suppliers (1) — (N) rm_purchases`
- `rm_products (1) — (N) rm_purchases`
- `rm_warehouses (1) — (N) rm_purchases`
- `rm_products (1) — (N) rm_movements`
- `rm_warehouses (1) — (N) rm_movements`
- `rm_employees (1) — (N) rm_movements`
- `rm_purchases (1) — (0..N) rm_movements` (optional: a movement may or may not originate from a purchase)

## Installation instructions
1. Install MySQL 8.x (locally or via Docker).
2. Connect with a client that supports local file loading, e.g.:
   ```
   mysql --local-infile=1 -u <user> -p
   ```
3. On the server, enable local file loading if it isn't already:
   ```sql
   SET GLOBAL local_infile = 1;
   ```

## Database creation
Run the DDL script to create the database and all tables:
```
mysql --local-infile=1 -u <user> -p < 03_ddl_schema.sql
```

## Data loading process
1. The raw Excel/CSV export was cleaned and deduplicated with a Python
   script (pandas), producing 8 clean CSV files under `csv_clean/`:
   `rm_cities.csv`, `rm_categories.csv`, `rm_suppliers.csv`,
   `rm_warehouses.csv`, `rm_products.csv`, `rm_employees.csv`,
   `rm_purchases.csv`, `rm_movements.csv`.
2. Update the file paths inside `04_load_data.sql` to point to your local
   `csv_clean/` folder, then run it:
   ```
   mysql --local-infile=1 -u <user> -p bd_jesus_villa_clan9 < 04_load_data.sql
   ```
3. Tables are loaded in dependency order (catalogs first, then entities that
   reference them, then transactional tables) so all foreign keys resolve
   correctly.

## SQL queries explanation
`06_consultas_negocio.sql` contains the 6 business queries requested:
1. **Available inventory per product** — latest recorded stock per product.
2. **Products stored per warehouse** — net quantity (IN − OUT) per warehouse/product.
3. **Total purchased per supplier** — total amount invested per supplier.
4. **Products with lowest stock** — top 5 products closest to running out.
5. **Top 5 most purchased products** — by total quantity purchased.
6. **Inventory value per city** — net stock valued at each product's most
   recent purchase price, aggregated by city.

`05_dml.sql` contains sample INSERT, UPDATE, and DELETE operations,
including a demonstration of the `ON DELETE RESTRICT` constraint blocking
deletion of a product that still has purchases or movements linked to it.

## Developer information
- **Full name:** Jesús Villa
- **Clan:** Clan 9
