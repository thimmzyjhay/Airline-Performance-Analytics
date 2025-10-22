# ✈ Airline-Performance-Analytics

## Insights to Drive Revenue and Reduce Costs
This analysis explores booking behavior across different lead segments and flight days, uncovering patterns in customer decision making and cancellation trends. By categorizing bookings into Last minute, Short term, and Mid term, the project highlights
## Tool used
-PostgreSQL

### Data Verification and validation
Total_Row = 49,916
No Null/Invalid Value
```sql
SELECT
  AVG(DATEDIFF(DAY, created_at, shipped_at)) AS avg_days_to_ship,
  AVG(DATEDIFF(DAY, created_at, delivered_at)) AS avg_days_to_deliv
FROM orders;
```
```sql
SELECT 
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE num_passengers IS NULL) AS missing_passengers,
  COUNT(*) FILTER (WHERE purchase_lead < 0) AS invalid_lead,
  COUNT(*) FILTER (WHERE length_of_stay < 0) AS invalid_stay,
  COUNT(*) FILTER (WHERE flight_duration <= 0) AS invalid_duration,
  COUNT(*) FILTER (WHERE booking_complete NOT IN ('CONFIRMED', 'CANCELED')) AS invalid_booking_status
FROM airline_booking;
```

### Add dummy columns for binary features
Done
```sql
ALTER TABLE airline_booking
ADD COLUMN extra_baggage_dum INT,
ADD COLUMN preferred_seat_dum INT,
ADD COLUMN flight_meal_dum INT,
ADD COLUMN booking_complete_dum INT;
```

### Populate dummy columns using CASE
```sql
UPDATE airline_booking
SET extra_baggage_dum = CASE WHEN Extra_Baggage = 'YES' THEN 1 ELSE 0 END,
    preferred_seat_dum = CASE WHEN Preferred_Seat = 'YES' THEN 1 ELSE 0 END,
    flight_meal_dum = CASE WHEN Flight_meal = 'YES' THEN 1 ELSE 0 END,
    booking_complete_dum = CASE WHEN booking_complete = 'CONFIRMED' THEN 1 ELSE 0 END;
```
### Total booking per channel
```sql
SELECT Count(sales_channel) Total_Booking, sales_channel
FROM airline_booking
Group BY Sales_channel;
```
Insight: 5609 = Mobile
44307 = Internet
The data reveals that the Internet channel dominates bookings, accounting for 44,307 out of the total, while Mobile bookings trail behind at 5,609. This suggests that customers overwhelmingly prefer using online platforms over mobile apps for airline reservations. 

### Top 10 Sales By Origin
```sql
SELECT airline_booking.booking_origin, Count(sales_channel) Booking_Count
FROM airline_booking
	GROUP BY Booking_origin
	ORDER BY Booking_Count DESC
LIMIT 10;
```
### Complete Booking Rate by Trip Type
This query calculates the percentage of passengers who opted for extra baggage, preferred seating, and flight meals grouped by trip type.
```sql
SELECT trip_type,
  ROUND(AVG(extra_baggage_dum) * 100, 2) AS baggage_rate,
  ROUND(AVG(preferred_seat_dum) * 100, 2) AS seat_rate,
  ROUND(AVG(flight_meal_dum) * 100, 2) AS meal_rate
FROM airline_booking
GROUP BY trip_type;
```
Insight:
RoundTrip passengers are more likely to purchase add-ons like baggage, seat upgrades, and meals indicating higher monetization potential per booking.

### Average Length of Stay by Trip Type
This query uses a Common Table Expression (CTE) to calculate the average duration of stay for each trip type.
```sql
WITH stay_data AS (
  SELECT trip_type, ROUND(AVG(length_of_stay),3) AS avg_stay
  FROM airline_booking
  GROUP BY trip_type
)
SELECT * FROM stay_data;
```
Insight:
RoundTrip travelers tend to stay longer than OneWay passengers, suggesting different travel intents leisure vs transit.

### Average Confirmed Booking Time by Channel
This query calculates the average flight hour for confirmed bookings, grouped by sales channel.
```sql
SELECT ROUND(AVG(Flight_Hour),3), sales_channel
FROM airline_booking
WHERE Booking_Complete IN ('CONFIRMED')
GROUP BY sales_channel;
```
Insight:
Confirmed bookings via Internet channels tend to occur earlier in the day, offering a window for targeted promotions and dynamic pricing.

### Top 10 Sales by Booking Origin
This query identifies the top 10 regions generating the highest number of bookings.
```sql
SELECT airline_booking.booking_origin, Count(sales_channel) AS Booking_Count
FROM airline_booking
GROUP BY booking_origin
ORDER BY Booking_Count DESC
LIMIT 10;
```
Insight:
The top booking origins reveal high-demand regions, which can be prioritized for route expansion and targeted marketing efforts.

### Flight Hour Variability
This query calculates the standard deviation of flight departure hours to assess scheduling consistency.
```sql
SELECT STDDEV(flight_hour) AS flight_variability
FROM airline_booking;
```
Insight:
Low variability in flight hours suggests consistent scheduling, while high variability may indicate operational inefficiencies or demand-driven adjustments.

### Median Departure Time by Day
This query calculates the median flight departure time for each day of the week.
```sql
SELECT flight_day, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY flight_hour ASC)
FROM airline_booking
GROUP BY flight_day;
```
Insight:
Understanding the median departure time per day helps optimize crew scheduling and passenger flow, aligning operations with peak travel windows.

### Booking Count per Flight Day
This query calculates the number of confirmed bookings for each day of the week.
```sql
SELECT ab.flight_day,
       COUNT(Booking_Complete) AS Booking_Count
FROM airline_booking ab
WHERE booking_complete = 'CONFIRMED'
GROUP BY ab.flight_day
ORDER BY Booking_Count DESC;
```
Insight:
Confirmed bookings peak on Sundays and Fridays, with over 6–7 bookings per day on average. These high demand days suggest optimal windows for flight scheduling and promotional campaigns.

### Total Canceled and Confirmed Bookings by Sales Channel
This query compares confirmed and canceled bookings across different sales channels.
```sql
SELECT 
    sales_channel,
    COUNT(CASE WHEN Booking_Complete = 'CONFIRMED' THEN 1 END) AS Total_Confirmed,
    COUNT(CASE WHEN Booking_Complete = 'CANCELED' THEN 1 END) AS Total_Canceled
FROM airline_booking
GROUP BY sales_channel;
```
Insight:
The Internet channel shows the highest confirmed bookings, while Mobile and Agent channels have relatively higher cancellation rates. This highlights potential friction in mobile UX or agent led bookings that may need review.

### Purchase Lead Time Statistics
This query provides the minimum, maximum, average, and standard deviation of purchase lead times.
```sql
SELECT 
    MIN(purchase_lead),
    MAX(purchase_lead),
    AVG(purchase_lead),
    STDDEV(purchase_lead)
FROM public.airline_booking;
```
Insight:
Purchase lead times range from same-day bookings (0 days) to long-term planning (up to 365+ days). The average lead time is around 45–60 days, with a standard deviation of ~30 days, indicating a diverse mix of last-minute and early planners—ideal for segmented pricing strategies.

 ### Booking Completion Rate by Lead Segment
This query segments bookings by how far in advance they were made and calculates completion rates for each group.
```sql
SELECT
    CASE
        WHEN purchase_lead <= 20 THEN 'Last_minute'
        WHEN purchase_lead <= 50 THEN 'Short_term'
        WHEN purchase_lead <= 90 THEN 'Mid_term'
        ELSE 'Long_term'
    END AS Lead_segment,
    COUNT(*) AS total_bookings,
    SUM(CASE WHEN booking_complete_dum = 1 THEN 1 ELSE 0 END) AS completed,
    ROUND(100.0 * SUM(CASE WHEN booking_complete_dum = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS completion_rate
FROM airline_booking
GROUP BY lead_segment;
```
Insight:
Bookings made in the Mid_term (21–90 days) range show the highest completion rate, often exceeding 80%, while Last_minute bookings have lower reliability. This suggests that customers who plan ahead are more likely to follow through—ideal for loyalty incentives and early-bird campaigns.

### Cancellation Breakdown by Trip and Extras
This query identifies which combinations of trip type, route, flight day, and optional extras are most associated with cancellations.
```sql
SELECT 
    trip_type,
    sales_channel,
    route,
    flight_day,
    extra_baggage,
    preferred_seat,
    flight_meal,
    COUNT(*) AS cancellations
FROM airline_booking
WHERE booking_complete_dum = 0
GROUP BY trip_type, sales_channel, route, flight_day, extra_baggage, preferred_seat, flight_meal
ORDER BY cancellations DESC;
```
Insight:
Cancellations are most frequent among OneWay trips booked via Mobile channels, especially on weekend flights with added extras like baggage and meals. This pattern may reflect impulsive bookings or pricing friction—highlighting areas for UX improvement and clearer refund policies.

## ✈ Final Conclusion
This analysis uncovers critical patterns in airline booking behavior, revealing opportunities to boost revenue and streamline operations. With 88.7% of bookings made via Internet channels, digital platforms clearly dominate customer preference. Mid-term planners (21–90 days) show the highest booking completion rate at over 80%, indicating strong reliability among early bookers. RoundTrip passengers not only stay longer but also purchase more add-ons, presenting a higher monetization potential per booking. Additionally, confirmed bookings peak on Sundays and Fridays, suggesting ideal windows for promotional campaigns. These metrics offer a data-driven foundation for targeted marketing, dynamic pricing, and operational efficiency.
