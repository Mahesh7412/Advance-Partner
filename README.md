#Schd Code
WITH driver AS (
  SELECT 
    TIMESTAMP(DATETIME(a.created_at, "Asia/Kolkata")) AS adv_requested,
   a.partner_number,
    a.requested_source,
    a.amount,
    round(a.deduction_fee,2) as deduction_fee,
    a.status,
    a.date,
    a.vehicle_number,
    CASE WHEN b.name = 'advance' THEN a.amount END AS advance_amount,
    CASE WHEN b.name = 'fuel_card' THEN a.amount END AS fuel_amount
  FROM `warehouse.wh_fieldops_driverextraexpence` AS a
  LEFT JOIN `lt-supply-app-dev.warehouse.wh_static_fieldops_driver_extra_expense_types` AS b 
    ON a.expence_type = b._id
),

book AS (
  SELECT 
    id,
    DATE(
      CASE 
        WHEN CAST(pick_up_at AS STRING) LIKE '%00:00:00+00%' 
        THEN TIMESTAMP_ADD(TIMESTAMP_ADD(TIMESTAMP_ADD(TIMESTAMP(pick_up_at), INTERVAL 1 DAY), INTERVAL 5 HOUR), INTERVAL 30 MINUTE)
        ELSE TIMESTAMP_ADD(TIMESTAMP_ADD(TIMESTAMP(pick_up_at), INTERVAL 5 HOUR), INTERVAL 30 MINUTE)
      END
    ) AS booking_date,
    city_code,
    trip_mode,
    lower(client_name) as client_name,
    'SCHD' AS business_type,
    CASE 
      WHEN extr_schd_adhoc_used = 0 THEN vehicle_owner_contact 
      ELSE final_driver_owner_contact 
    END AS partner_number,
    lower(CASE 
      WHEN extr_schd_adhoc_used = 0 THEN vehicle_owner_name 
      ELSE final_driver_owner_name 
    END) AS partner_name,
    extr_used_vehicle_number as vehicle_number,extr_final_unfulfilled,is_active
  FROM `warehouse.wh_fieldops_scheduled_bookings`
WHERE extr_final_unfulfilled = 0 AND is_active = TRUE
),

joined_data AS (
  SELECT 
    a.*, 
    b.adv_requested,
    b.requested_source,
    b.amount,
    b.deduction_fee,
    b.status,
    b.date,
    b.advance_amount,
    b.fuel_amount,
  FROM book AS a
  LEFT JOIN driver AS b
    ON FORMAT_TIMESTAMP('%b-%Y', a.booking_date) = FORMAT_TIMESTAMP('%b-%Y', TIMESTAMP(DATETIME(b.date, "Asia/Kolkata")))
    AND a.vehicle_number = b.vehicle_number
    AND CAST(a.partner_number AS FLOAT64) = b.partner_number
)
SELECT * ,REPLACE(CAST(partner_number AS STRING), '.0', '') as partner_num
FROM joined_data


