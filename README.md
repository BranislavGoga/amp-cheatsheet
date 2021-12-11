# Statistics

## Number of Orders per day
```
SELECT date_trunc('day', "accepted_by_packer_at") AS day, COUNT(*) AS finished_orders
FROM "multi_order" AS m
JOIN "order_in_multi_order" ON "multi_order_id" = "m"."id"
WHERE "state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-10-01 00:00:00' 
AND "accepted_by_packer_at" <= '2021-10-31 23:59:59' 
GROUP BY day
ORDER BY "day"
```


## Number of MultiOrders per day
```
SELECT date_trunc('day', "accepted_by_packer_at") AS day, COUNT(*) AS finished_multi_orders
FROM "multi_order" AS m
WHERE "state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-10-01 00:00:00' 
AND "accepted_by_packer_at" <= '2021-10-31 23:59:59' 
GROUP BY day
ORDER BY "day"
```

## Number of picks per day:
```
SELECT date_trunc('day', "accepted_by_packer_at") AS day, SUM(sub_order.quantity) AS picked_quantity
FROM "sub_order"
JOIN "picker_command" ON "picker_command"."id" = "sub_order"."picker_command_id"
JOIN "sub_multi_order" ON "sub_multi_order"."id" = "picker_command"."sub_multi_order_id"
JOIN "multi_order" ON "multi_order"."id" = "sub_multi_order"."multi_order_id"
WHERE "sub_multi_order"."state" = 'flushed_to_runner'
AND "multi_order"."accepted_by_packer_at" >= '2021-10-01 00:00:00'
AND "multi_order"."accepted_by_packer_at" <= '2021-10-31 23:59:59'
GROUP BY "day"
ORDER BY "day"
```

## Average pick time per sector
```
SELECT sector_id, wms_sector_id, AVG(finished_at - accepted_at), COUNT(p.id) AS count, MIN(finished_at - accepted_at) AS min_picker_cycle_time, MAX(finished_at - accepted_at) AS max_picker_cycle_time
FROM picker_command AS p
JOIN external_command AS e ON picker_command_id = p.id
JOIN sub_multi_order AS s ON p.sub_multi_order_id = s.id
JOIN sector AS c ON c.id = s.sector_id
WHERE p.finished_at >= '2021-11-19 00:00:00' 
AND p.finished_at <= '2021-11-19 23:59:59'
AND e.type = 'picking_unit_ready'
GROUP BY sector_id, wms_sector_id
ORDER BY sector_id
```

## Average pick time per sertor per hour of the day
```
SELECT date_trunc('hour', finished_at) AS hour, sector_id, wms_sector_id, AVG(finished_at - accepted_at) AS avg_picker_cycle_time, COUNT(p.id) AS count, MIN(finished_at - accepted_at) AS min_picker_cycle_time, MAX(finished_at - accepted_at) AS max_picker_cycle_time
FROM picker_command AS p
JOIN external_command AS e ON picker_command_id = p.id
JOIN sub_multi_order AS s ON p.sub_multi_order_id = s.id
JOIN sector AS c ON c.id = s.sector_id
WHERE p.finished_at >= '2021-11-19 00:00:00' 
AND p.finished_at <= '2021-11-19 23:59:59'
AND e.type = 'picking_unit_ready'
GROUP BY hour, sector_id, wms_sector_id
ORDER BY hour, sector_id
```

## List sub-multi-orders stored in slots
Picker commands = (sub multi orders, 1:1 relationship) stored in slots = there is store command for them, but not flush command = there is only one matrix sorter command.

```
SELECT * 
FROM (
SELECT picker_command_id, sector_id, COUNT(*) AS matrix_sorter_command_count
FROM matrix_sorter_command
GROUP BY picker_command_id, sector_id
ORDER BY matrix_sorter_command_count) AS tmp
WHERE matrix_sorter_command_count < 2
```

## Number of full slots per sector
```
SELECT sector_id, COUNT(*) full_slot_count
FROM (
SELECT picker_command_id, sector_id, COUNT(*) AS matrix_sorter_command_count
FROM matrix_sorter_command
GROUP BY picker_command_id, sector_id
ORDER BY matrix_sorter_command_count) AS tmp
WHERE matrix_sorter_command_count < 2
GROUP BY sector_id
ORDER BY sector_id
```

## List of Orders fulfilled in a day
```
SELECT date_trunc('day', "accepted_by_packer_at") AS day, wms_order_id, cell_id, packing_delivery_unit, fulfillment_start_at, accepted_by_packer_at
FROM "multi_order" AS m
JOIN "order_in_multi_order" AS oim ON "multi_order_id" = "m"."id"
JOIN "order" AS o ON "oim"."order_id" = "o"."id"
WHERE "m"."state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-10-01 00:00:00' 
AND "accepted_by_packer_at" <= '2021-10-31 23:59:59' 
ORDER BY "accepted_by_packer_at"
```

As the query may become big and not fit into browser, it may be convenient to run it from shell:
```
YEAR=2021
MONTH=11

DBSERVER=10.5.185.13
DBUSER=postgres
export PGPASSWORD=matrixpick1

SQL="SELECT date_trunc('day', accepted_by_packer_at) AS day, wms_order_id, cell_id, packing_delivery_unit, fulfillment_start_at, accepted_by_packer_at 
FROM multi_order AS m
JOIN order_in_multi_order AS oim ON multi_order_id = m.id
JOIN \"order\" AS o ON oim.order_id = o.id
WHERE m.state = 'accepted_in_packing' 
AND accepted_by_packer_at >= '$YEAR-$MONTH-01 00:00:00' 
AND accepted_by_packer_at <= '$YEAR-$MONTH-01 00:00:00'::timestamp + '1 month'::interval 
ORDER BY accepted_by_packer_at"

FILENAME="$YEAR-$MONTH-order-list.csv"
psql -h $DBSERVER -U $DBUSER -w -c "$SQL" --no-align --csv -F=, matrixpick-core-planner > $FILENAME
echo "Output written to file '$FILENAME'"
```

## Average number of Orders in MultiOrder:
```
SELECT AVG(c)
FROM (
SELECT m.id, COUNT(*) as c
FROM "multi_order" AS m
JOIN "order_in_multi_order" ON "multi_order_id" = "m"."id"
WHERE "state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-11-02 00:00:00' 
AND "accepted_by_packer_at" <= '2021-11-02 23:59:59'
GROUP BY "m"."id" 
) AS tmp
```

## Average number of Items (quantity) in Orders fulfilled on specific date
```
SELECT AVG(quantity)
FROM (
SELECT o.id, SUM(s.quantity) AS quantity
FROM "multi_order" AS m
JOIN "order_in_multi_order" AS oim ON "oim"."multi_order_id" = "m"."id"
JOIN "order" AS o ON "o"."id" = "oim"."order_id"
JOIN "sub_order" AS s ON "s"."order_id" = "o"."id"
WHERE "m"."state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-11-05 00:00:00' 
AND "accepted_by_packer_at" <= '2021-11-05 23:59:59'
GROUP BY "o"."id"
) as tmp
```

## Average number of stops per MultiOrder:
```
SELECT AVG(c)
FROM (
SELECT m.id, COUNT(*) as c
FROM "multi_order" AS m
JOIN "sub_multi_order" AS smo ON "multi_order_id" = "m"."id"
WHERE "m"."state" = 'accepted_in_packing' 
AND "accepted_by_packer_at" >= '2021-11-02 14:00:00' 
AND "accepted_by_packer_at" <= '2021-11-02 23:59:59'
GROUP BY "m"."id" 
) tmp
```

## Average time Runner was waiting at packing since EMANS accepted packing request:

```
SELECT AVG("accepted_by_packer_at" - "accepted_by_wms"), MIN("accepted_by_packer_at" - "accepted_by_wms"), MAX("accepted_by_packer_at" - "accepted_by_wms")
FROM (
SELECT m.id, e.accepted_at AS accepted_by_wms, m.accepted_by_packer_at + INTERVAL '1 hour' AS accepted_by_packer_at
FROM multi_order AS m
JOIN external_command AS e ON "e"."multiorder_id" = "m"."id"
WHERE "e"."type" = 'pack'
AND "m"."accepted_by_packer_at" >= '2021-11-02 00:00'
AND "m"."accepted_by_packer_at" < '2021-11-02 23:59'
) AS tmp
```

## Average time Runner was waiting at packing since we started to try to send the packing request to EMANS:
```
SELECT AVG("accepted_by_packer_at" - "accepted_by_wms"), MIN("accepted_by_packer_at" - "accepted_by_wms"), MAX("accepted_by_packer_at" - "accepted_by_wms")
FROM (
SELECT m.id, e.created_at AS accepted_by_wms, m.accepted_by_packer_at AS accepted_by_packer_at
FROM multi_order AS m
JOIN external_command AS e ON "e"."multiorder_id" = "m"."id"
WHERE "e"."type" = 'pack'
AND "m"."accepted_by_packer_at" >= '2021-11-02 00:00'
AND "m"."accepted_by_packer_at" < '2021-11-02 23:59'
) AS tmp
```

## WMS Order IDs for MultiOrder
```
SELECT multi_order_id, order_id, position_in_matrix, wms_order_id
FROM multi_order AS m
JOIN order_in_multi_order AS oim ON "m"."id" = "oim"."multi_order_id"
JOIN "order" AS o ON "o"."id" = "oim"."order_id"
WHERE "m"."id"=12831 
```

## Boxes assigned to MultiOrder

```
SELECT DISTINCT b.barcode, b.fleet_command_id, b.position_in_matrix, b.runner_id
FROM box AS b
JOIN fleet_command AS fc ON "b"."fleet_command_id" = "fc"."id"
JOIN fleet_command_station AS fcs ON "fcs"."fleet_command_id" = "fc"."id"
JOIN picker_command AS pc ON "pc"."id" = "fcs"."picker_command_id"
JOIN sub_multi_order AS smo ON "smo"."id" = "pc"."sub_multi_order_id"
WHERE "smo"."multi_order_id" = '20405'
ORDER BY "b"."position_in_matrix"
```

All box assignments (after every FleetContinue) - if everything is ok, only duplicates
```
SELECT DISTINCT b.*
FROM box AS b
JOIN fleet_command AS fc ON "b"."fleet_command_id" = "fc"."id"
JOIN fleet_command_station AS fcs ON "fcs"."fleet_command_id" = "fc"."id"
JOIN picker_command AS pc ON "pc"."id" = "fcs"."picker_command_id"
JOIN sub_multi_order AS smo ON "smo"."id" = "pc"."sub_multi_order_id"
WHERE "smo"."multi_order_id" = '12831'
ORDER BY "b"."position_in_matrix"
```

##Orders and boxes on Runner
```
SELECT DISTINCT "order".wms_order_id, "order".cell_id, box.barcode 
FROM sub_multi_order
JOIN picker_command ON picker_command.sub_multi_order_id = sub_multi_order.id
JOIN fleet_command_station ON fleet_command_station.picker_command_id = picker_command.id
JOIN fleet_command ON fleet_command_station.fleet_command_id = fleet_command.id
JOIN box ON box.fleet_command_id = fleet_command.id
JOIN order_in_multi_order ON (order_in_multi_order.multi_order_id = sub_multi_order.multi_order_id AND order_in_multi_order.position_in_matrix = box.position_in_matrix)
JOIN "order" ON "order".id = order_in_multi_order.id
WHERE sub_multi_order.multi_order_id = 11
```

### MultiOrder and SubMultiOrders assigned to Runner
```
SELECT *
FROM runner AS r
JOIN fleet_command AS fc ON "fc"."runner_id" = "r"."id"
JOIN fleet_command_station AS fcs ON "fcs"."fleet_command_id" = "fc"."id"
JOIN picker_command AS pc ON "pc"."id" = "fcs"."picker_command_id"
JOIN sub_multi_order AS smo ON "smo"."id" = "pc"."sub_multi_order_id"
WHERE "r"."id" = '19' AND "fc"."state" != 'success'
```

## Stop all locked multi-orders
Stop all MultiOrders that are locked. No new Runner shold be assigned / no MultiOrder started.

```
UPDATE "multi_order" 
SET "state" = 'failed'
WHERE "state" = 'locked';
```

Then put them back (**ONLY if there were no other `failed` when running previous script**):

```
UPDATE "multi_order" 
SET "state" = 'locked'
WHERE "state" = 'failed';
```

## Remove failed multiorders and retry
Removes all failed multiorders and mark all connected orders as new to forece core to retry

```
--- 
--- Alter 
---
WITH R as (
    SELECT 
        M.id as m_id, M.state as m_state,
        O.id as o_id, O.state as o_state
    FROM "multi_order" M
        INNER JOIN "order_in_multi_order" OM ON OM.multi_order_id = M.id
        INNER JOIN "order" O ON OM.order_id = O.id
    WHERE 
        M.state='failed' AND O.state='locked'
)
UPDATE
    "order" AS O
SET
    state = 'new'
FROM R
WHERE 
    O.id = R.o_id AND R.m_state = 'failed' and R.o_state = 'locked';

---
--- Clean up
---
DELETE FROM  
    "picker_command" P
USING 
    "sub_multi_order" SM,
    "multi_order" M
WHERE 
    SM.multi_order_id = M.id AND 
    P.sub_multi_
    "multi_order" M
WHERE 
    SM.multi_order_id = M.id AND 
    M.state='failed';

---
DELETE FROM
    "external_command" EC
USING 
    "multi_order" M
WHERE 
    EC.multiorder_id = M.id AND
    M.state='failed';

---
DELETE FROM 
    "order_in_multi_order" OM
USING
    "multi_order" M
WHERE 
    M.state='failed' AND OM.multi_order_id = M.id;

---
DELETE FROM 
    "multi_order"
WHERE 
    state='failed';
DELETE FROM 
    "sub_multi_order" SM 
USING 
    "multi_order" M
WHERE 
    SM.multi_order_id = M.id AND 
    M.state='failed';

---
DELETE FROM
    "external_command" EC
USING 
    "multi_order" M
WHERE 
    EC.multiorder_id = M.id AND
    M.state='failed';

---
DELETE FROM 
    "order_in_multi_order" OM
USING
    "multi_order" M
WHERE 
    M.state='failed' AND OM.multi_order_id = M.id;

---
DELETE FROM 
    "multi_order"
WHERE 
    state='failed';
```

# Special Queries

## Find Order, that adds the most (maximizes) average quantity per stop to already chosen Orders
```
SELECT "o"."id" AS oid, AVG(quantity) AS avg
FROM sub_order AS s, "order" AS o
WHERE "s"."order_id" <= 2000 OR "s"."order_id" = "o"."id"
GROUP BY o.id
ORDER BY avg DESC
```

## Find Orders with minimal number of sectors
```
SELECT order_id, count(sub_order.sector_id) AS sector_count 
FROM "order"
JOIN "sub_order" ON "sub_order"."order_id" = "order"."id"
WHERE "order".state = 'new' 
GROUP BY order_id 
ORDER BY sector_count
```
