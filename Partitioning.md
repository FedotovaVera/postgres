
```
demo=# CREATE TABLE bookings.ticket_flights_part (
demo(#                 ticket_no character(13) NOT NULL,
demo(#                 flight_id integer NOT NULL,
demo(#                 fare_conditions character varying(10) NOT NULL,
demo(#                 amount numeric(10,2) NOT NULL,
demo(#                 CONSTRAINT ticket_flights_amount_check CHECK ((amount >= (0)::numeric)),
demo(#                 CONSTRAINT ticket_flights_fare_conditions_check CHECK (((fare_conditions)::text = ANY (ARRAY[('Economy'::character varying)::text, ('Comfort'::character varying)::text, ('Business'::character varying)::text])))
demo(#             )
demo-#             partition by hash(ticket_no);
```
```
demo=#     create table ticket_flights_1 partition of ticket_flights_part for values with (modulus 5, remainder 0);
CREATE TABLE
demo=#     create table ticket_flights_2 partition of ticket_flights_part for values with (modulus 5, remainder 1);
CREATE TABLE
demo=#     create table ticket_flights_3 partition of ticket_flights_part for values with (modulus 5, remainder 2);
CREATE TABLE
demo=#     create table ticket_flights_4 partition of ticket_flights_part for values with (modulus 5, remainder 3);
CREATE TABLE
demo=#     create table ticket_flights_5 partition of ticket_flights_part for values with (modulus 5, remainder 4);
CREATE TABLE
demo=# INSERT INTO bookings.ticket_flights_part SELECT * FROM bookings.ticket_flights;
INSERT 0 2360335
```
```
demo=# select count(*) from ticket_flights;
  count
---------
 2360335
(1 row)

demo=# select count(*) from ticket_flights_part;
  count
---------
 2360335
(1 row)

demo=# select count(*) from ticket_flights_1;
 count
--------
 471422
(1 row)
```

