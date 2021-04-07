# Instructor script
## Content
- [Create database tables](#create-database-tables)
- [Create Data Model with CDS](#create-data-model-with-cds)
- [Create CDS projection](#create-cds-projection)
- [Enrich projection with UI Metadata](#enrich-projection-with-ui-metadata)
- [Define and expose service](#define-and-expose-service)
## Create database tables
1. Create package ZRAP_TRAVEL_SFMX
2. Add to favorites the parent package
### Travel database table
3. Create travel database table ZRAP_ATRAV_SFMX
4. Replace code
    ```ABAP
    @EndUserText.label : 'Travel data'
    @AbapCatalog.enhancementCategory : #NOT_EXTENSIBLE
    @AbapCatalog.tableCategory : #TRANSPARENT
    @AbapCatalog.deliveryClass : #A
    @AbapCatalog.dataMaintenance : #RESTRICTED
    define table zrap_atrav_sfmx {
      key client            : mandt not null;
      key travel_uuid       : sysuuid_x16 not null;
      travel_id             : /dmo/travel_id;
      agency_id             : /dmo/agency_id;
      customer_id           : /dmo/customer_id;
      begin_date            : /dmo/begin_date;
      end_date              : /dmo/end_date;
      @Semantics.amount.currencyCode : 'zrap_atrav_sfmx.currency_code'
      booking_fee           : /dmo/booking_fee;
      @Semantics.amount.currencyCode : 'zrap_atrav_sfmx.currency_code'
      total_price           : /dmo/total_price;
      currency_code         : /dmo/currency_code;
      description           : /dmo/description;
      overall_status        : /dmo/overall_status;
      created_by            : syuname;
      created_at            : timestampl;
      last_changed_by       : syuname;
      last_changed_at       : timestampl;
      local_last_changed_at : timestampl;
    }
    ```
### Booking database table
5. Create booking database table ZRAP_ABOOK_SFMX
6. Replace code
    ```ABAP
    @EndUserText.label : 'Booking data'
    @AbapCatalog.enhancementCategory : #NOT_EXTENSIBLE
    @AbapCatalog.tableCategory : #TRANSPARENT
    @AbapCatalog.deliveryClass : #A
    @AbapCatalog.dataMaintenance : #RESTRICTED
    define table zrap_abook_sfmx {
      key client            : mandt not null;
      key booking_uuid      : sysuuid_x16 not null;
      travel_uuid           : sysuuid_x16 not null;
      booking_id            : /dmo/booking_id;
      booking_date          : /dmo/booking_date;
      customer_id           : /dmo/customer_id;
      carrier_id            : /dmo/carrier_id;
      connection_id         : /dmo/connection_id;
      flight_date           : /dmo/flight_date;
      @Semantics.amount.currencyCode : 'zrap_abook_sfmx.currency_code'
      flight_price          : /dmo/flight_price;
      currency_code         : /dmo/currency_code;
      created_by            : syuname;
      last_changed_by       : syuname;
      local_last_changed_at : timestampl;
    }
    ```  
7. Save and activate.
8. Data preview (empty)
### Table data generator
9. Create class ZCL_GENERATE_DEMO_DATA_SFMX
10. Replace code (Remember to refactor methods into delete and populate)
    ```ABAP
    METHOD if_oo_adt_classrun~main.

    " delete existing entries in the database table
    DELETE FROM zrap_atrav_sfmx.
    DELETE FROM zrap_abook_sfmx.

    " insert travel demo data
    INSERT zrap_atrav_sfmx FROM (
        SELECT FROM 
            /dmo/travel
        FIELDS
            uuid(  )      AS travel_uuid,
            travel_id     AS travel_id,
            agency_id     AS agency_id,
            customer_id   AS customer_id,
            begin_date    AS begin_date,
            end_date      AS end_date,
            booking_fee   AS booking_fee,
            total_price   AS total_price,
            currency_code AS currency_code,
            description   AS description,
            CASE status
              WHEN 'B' THEN 'A' " accepted
              WHEN 'X' THEN 'X' " cancelled
              ELSE 'O'          " open
            END           AS overall_status,
            createdby     AS created_by,
            createdat     AS created_at,
            lastchangedby AS last_changed_by,
            lastchangedat AS last_changed_at,
            lastchangedat AS local_last_changed_at
        ORDER BY 
            travel_id UP TO 200 ROWS
      ).
    COMMIT WORK.

    out->write( 'Travel demo data inserted.').

    " insert booking demo data
    INSERT zrap_abook_sfmx FROM (
        SELECT FROM   
            /dmo/booking AS booking
        JOIN 
            zrap_atrav_sfmx AS z ON   
            booking~travel_id = z~travel_id
          FIELDS
            uuid( )                 AS booking_uuid,
            z~travel_uuid           AS travel_uuid,
            booking~booking_id      AS booking_id,
            booking~booking_date    AS booking_date,
            booking~customer_id     AS customer_id,
            booking~carrier_id      AS carrier_id,
            booking~connection_id   AS connection_id,
            booking~flight_date     AS flight_date,
            booking~flight_price    AS flight_price,
            booking~currency_code   AS currency_code,
            z~created_by            AS created_by,
            z~last_changed_by       AS last_changed_by,
            z~last_changed_at       AS local_last_changed_by
      ).
    COMMIT WORK.

    out->write( 'Booking demo data inserted.').
    ENDMETHOD.
    ```
11. Save and activate.
12. Press F9 to run in console
13. Preview tables

## Create Data Model with CDS
### Travel CDS
1. Create travel CDS ZI_RAP_TRAVEL_SFMX (any template, will replace)
2. Replace code
    ```ABAP
    @AccessControl.authorizationCheck: #CHECK
    @EndUserText.label: 'Travel BO view'
    define view entity zi_rap_travel_sfmx
    as select from zrap_atrav_sfmx as Travel

    association [0..*] to zi_rap_booking_sfmx as _Booking on $projection.TravelUUID = _Booking.TravelUUID

    association [0..1] to /DMO/I_Agency       as _Agency   on $projection.AgencyID = _Agency.AgencyID
    association [0..1] to /DMO/I_Customer     as _Customer on $projection.CustomerID = _Customer.CustomerID
    association [0..1] to I_Currency          as _Currency on $projection.CurrencyCode = _Currency.Currency  
    {
    key travel_uuid           as TravelUUID,
        travel_id             as TravelID,
        agency_id             as AgencyID,
        customer_id           as CustomerID,
        begin_date            as BeginDate,
        end_date              as EndDate,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        booking_fee           as BookingFee,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        total_price           as TotalPrice,
        currency_code         as CurrencyCode,
        description           as Description,
        overall_status        as TravelStatus,
        @Semantics.user.createdBy: true
        created_by            as CreatedBy,
        @Semantics.systemDateTime.createdAt: true
        created_at            as CreatedAt,
        @Semantics.user.lastChangedBy: true
        last_changed_by       as LastChangedBy,
        @Semantics.systemDateTime.lastChangedAt: true
        last_changed_at       as LastChangedAt,
        @Semantics.systemDateTime.localInstanceLastChangedAt: true
        local_last_changed_at as LocalLastChangedAt,

        /* associations */
        _Booking,
        _Agency,
        _Customer,
        _Currency      
    }
    ```
3. Cannot activate, just save
### Booking CDS
4. Create booking CDS ZI_RAP_BOOKING_SFMX (any template, will replace)
    ```ABAP
    @AccessControl.authorizationCheck: #CHECK
    @EndUserText.label: 'Booking BO view'
    define view entity zi_rap_booking_sfmx
    as select from zrap_abook_sfmx as Booking

    association [1..1] to zi_rap_travel_sfmx        as _Travel     on  $projection.TravelUUID = _Travel.TravelUUID
    
    association [1..1] to /DMO/I_Customer           as _Customer   on  $projection.CustomerID   = _Customer.CustomerID
    association [1..1] to /DMO/I_Carrier            as _Carrier    on  $projection.CarrierID    = _Carrier.AirlineID
    association [1..1] to /DMO/I_Connection         as _Connection on  $projection.CarrierID    = _Connection.AirlineID
                                                                    and $projection.ConnectionID = _Connection.ConnectionID
    association [1..1] to /DMO/I_Flight             as _Flight     on  $projection.CarrierID    = _Flight.AirlineID
                                                                    and $projection.ConnectionID = _Flight.ConnectionID
                                                                    and $projection.FlightDate   = _Flight.FlightDate
    association [0..1] to I_Currency                as _Currency   on $projection.CurrencyCode    = _Currency.Currency    
    {
    key booking_uuid          as BookingUUID,
        travel_uuid           as TravelUUID,
        booking_id            as BookingID,
        booking_date          as BookingDate,
        customer_id           as CustomerID,
        carrier_id            as CarrierID,
        connection_id         as ConnectionID,
        flight_date           as FlightDate,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        flight_price          as FlightPrice,
        currency_code         as CurrencyCode,
        @Semantics.user.createdBy: true
        created_by            as CreatedBy,
        @Semantics.user.lastChangedBy: true
        last_changed_by       as LastChangedBy,
        @Semantics.systemDateTime.localInstanceLastChangedAt: true
        local_last_changed_at as LocalLastChangedAt,

        /* associations */
        _Travel,
        _Customer,
        _Carrier,
        _Connection,
        _Flight,
        _Currency
    }
    ```
5. Save and activate all objects.
6. Preview data

### Define composition tree
7. Edit ZI_RAP_TRAVEL_SFMX.
    - Add the root keyword
        ```ABAP
        define root view entity zi_rap_travel_sfmx
        ```
    - Change the association to a composition
        ```ABAP
        composition [0..*] of zi_rap_booking_sfmx as _Booking 
        ```
8. Edit ZI_RAP_BOOKING_SFMX
    - Change the association to association to parent
        ```ABAP
        association to parent zi_rap_travel_sfmx as _Travel on  $projection.TravelUUID = _Travel.TravelUUID
        ```
9. These changes will have no impact on data.

## Create CDS projection
### Travel CDS projection
1. Create projection view ZC_RAP_TRAVEL_SFMX
2. Modify code
    ```ABAP
    @EndUserText.label: 'Travel BO projection view'
    @AccessControl.authorizationCheck: #CHECK
    @Search.searchable: true
    @Metadata.allowExtensions: true

    define root view entity zc_rap_travel_sfmx
    as projection on zi_rap_travel_sfmx as Travel
    {
    key TravelUUID,
        @Search.defaultSearchElement: true
        TravelID,
        @Consumption.valueHelpDefinition: [{ entity: { name: '/DMO/I_Agency', element: 'AgencyID'} }]
        @ObjectModel.text.element: ['AgencyName']
        @Search.defaultSearchElement: true
        AgencyID,
        _Agency.Name       as AgencyName,
        @Consumption.valueHelpDefinition: [{ entity: { name: '/DMO/I_Customer', element: 'CustomerID'} }]
        @ObjectModel.text.element: ['CustomerName']
        @Search.defaultSearchElement: true
        CustomerID,
        _Customer.LastName as CustomerName,
        BeginDate,
        EndDate,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        BookingFee,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        TotalPrice,
        @Consumption.valueHelpDefinition: [{ entity: { name: 'I_Currency', element: 'Currency'} }]
        CurrencyCode,
        Description,
        TravelStatus,
        LastChangedAt,
        LocalLastChangedAt,

        /* Associations */
        _Agency,
        _Booking : redirected to composition child zc_rap_booking_sfmx,
        _Currency,
        _Customer   
    }
    ```
3. Save without activating.
### Booking CDS projection
4. Create projection view ZC_RAP_BOOKING_SFMX
5. Modify code
    ```ABAP
    @EndUserText.label: 'Booking BO projection view'
    @AccessControl.authorizationCheck: #CHECK
    @Search.searchable: true
    @Metadata.allowExtensions: true

    define view entity zc_rap_booking_sfmx
    as projection on zi_rap_booking_sfmx as Booking
    {
    key BookingUUID,
        TravelUUID,
        @Search.defaultSearchElement: true
        BookingID,
        BookingDate,
        @Consumption.valueHelpDefinition: [{ entity : {name: '/DMO/I_Customer', element: 'CustomerID'  } }]
        @ObjectModel.text.element: ['CustomerName']
        @Search.defaultSearchElement: true
        CustomerID,
        _Customer.LastName as CustomerName,
        @Consumption.valueHelpDefinition: [{entity: {name: '/DMO/I_Carrier', element: 'AirlineID' }}]
        @ObjectModel.text.element: ['CarrierName']
        CarrierID,
        _Carrier.Name      as CarrierName,
        @Consumption.valueHelpDefinition: [ {entity: {name: '/DMO/I_Flight', element: 'ConnectionID'},
                                            additionalBinding: [ { localElement: 'CarrierID',    element: 'AirlineID' },
                                                                    { localElement: 'FlightDate',   element: 'FlightDate',   usage: #RESULT},
                                                                    { localElement: 'FlightPrice',  element: 'Price',        usage: #RESULT },
                                                                    { localElement: 'CurrencyCode', element: 'CurrencyCode', usage: #RESULT } ] } ]
        ConnectionID,
        FlightDate,
        @Semantics.amount.currencyCode: 'CurrencyCode'
        FlightPrice,
        @Consumption.valueHelpDefinition: [{entity: {name: 'I_Currency', element: 'Currency' }}]
        CurrencyCode,
        LocalLastChangedAt,

        /* associations */
        _Travel : redirected to parent zc_rap_travel_sfmx,
        _Customer,
        _Carrier,
        _Connection,
        _Flight
    }
    ```
### Activate Data Model 
6. Activate all objects
7. Data preview for each CDS ZC_RAP_TRAVEL_SFMX and ZC_RAP_BOOKING_SFMX

## Enrich projection with UI Metadata
### Annotate Travel view
1. Create metadata extension ZC_RAP_TRAVEL_SFMX (annotate view template)
2. Show that after adding ZC_RAP_TRAVEL_SFMX you can use Ctrl+Space to show all elements
3. Modify code
    ```ABAP
    @Metadata.layer: #CORE
    @UI: {
    headerInfo: { typeName: 'Travel',
                    typeNamePlural: 'Travels',
                    title: { type: #STANDARD, label: 'Travel', value: 'TravelID' } },
    presentationVariant: [{ sortOrder: [{ by: 'TravelID', direction:  #DESC }] }] }

    annotate view zc_rap_travel_sfmx with
    {
    @UI.facet: [ { id:              'Travel',
                    purpose:         #STANDARD,
                    type:            #IDENTIFICATION_REFERENCE,
                    label:           'Travel',
                    position:        10 },
                { id:              'Booking',
                    purpose:         #STANDARD,
                    type:            #LINEITEM_REFERENCE,
                    label:           'Booking',
                    position:        20,
                    targetElement:   '_Booking'} ]  

    @UI:{ identification: [{ position: 1, label: 'Travel UUID' }] }
    TravelUUID;

    @UI: {  lineItem:       [ { position: 10 } ],
            identification: [ { position: 10 } ],
            selectionField: [ { position: 10 } ] }  
    TravelID;

    @UI: {  lineItem:       [ { position: 20 } ],
            identification: [ { position: 20 } ],
            selectionField: [ { position: 20 } ] }  
    AgencyID;

    @UI: {  lineItem:       [ { position: 30 } ],
            identification: [ { position: 30 } ],
            selectionField: [ { position: 30 } ] }  
    CustomerID;

    @UI: {  lineItem:       [ { position: 40 } ],
            identification: [ { position: 40 } ] }  
    BeginDate;

    @UI: {  lineItem:       [ { position: 50 } ],
            identification: [ { position: 50 } ] }   
    EndDate;

    @UI: {  lineItem:       [ { position: 60 } ],
            identification: [ { position: 60 } ] }   
    BookingFee;

    @UI: {  lineItem:       [ { position: 70 } ],
            identification: [ { position: 70 } ] }   
    TotalPrice;

    @UI: {  lineItem:       [ { position: 80 } ],
            identification: [ { position: 80 } ] }   
    Description;

    @UI: {  lineItem:       [ { position: 90 } ],
            identification: [ { position: 90 } ] }   
    TravelStatus;

    @UI.hidden: true
    LastChangedAt;

    @UI.hidden: true
    LocalLastChangedAt;
    }    
    ```
### Annotate Booking view
<pre>
- @Metadata.layer
    - #CORE: Application provider
    - #CUSTOMER: Lower hierarchy
</pre>
4. Create metadata extension ZC_RAP_BOOKING_SFMX (annotate view template)
5. Show that you can bring up elements
6. Modify code
    ```ABAP
    @Metadata.layer: #CORE
    @UI: {
    headerInfo: { typeName: 'Booking',
                    typeNamePlural: 'Bookings',
                    title: { type: #STANDARD, value: 'BookingID' } } }

    annotate view zc_rap_booking_sfmx
        with 
    {
    @UI.facet: [ { id:            'Booking',
                    purpose:       #STANDARD,
                    type:          #IDENTIFICATION_REFERENCE,
                    label:         'Booking',
                    position:      10 }  ]

    @UI: { identification: [ { position: 10, label: 'Booking UUID'  } ] }
    BookingUUID;

    @UI.hidden: true
    TravelUUID;

    @UI: { lineItem:       [ { position: 20 } ],
            identification: [ { position: 20 } ] }
    BookingID;

    @UI: { lineItem:       [ { position: 30 } ],
            identification: [ { position: 30 } ] }
    BookingDate;

    @UI: { lineItem:       [ { position: 40 } ],
            identification: [ { position: 40 } ] }
    CustomerID;

    @UI: { lineItem:       [ { position: 50 } ],
            identification: [ { position: 50 } ] }
    CarrierID;

    @UI: { lineItem:       [ { position: 60 } ],
            identification: [ { position: 60 } ] }
    ConnectionID;

    @UI: { lineItem:       [ { position: 70 } ],
            identification: [ { position: 70 } ] }
    FlightDate;

    @UI: { lineItem:       [ { position: 80 } ],
            identification: [ { position: 80 } ] }
    FlightPrice;

    @UI.hidden: true
    LocalLastChangedAt;
    }
    ```

## Define and expose service
### Create service definition
1. Create service definition ZC_RAP_TRAVEL_SFMX by right clicking on travel data projection
2. Modify code and expose tables and catalogs
    ```ABAP CDS
    @EndUserText.label: 'Serv Definition for Travel App'
    define service zui_rap_travel_sfmx {
    expose zc_rap_travel_sfmx as Travel;
    expose zc_rap_booking_sfmx as Booking;
    expose /DMO/I_Agency as Agency;
    expose /DMO/I_Customer as Customer;
    expose /DMO/I_Flight as Flight;
    expose /DMO/I_Carrier as Carrier;
    expose /DMO/I_Connection as Connection;
    expose /DMO/I_Airport as Airport;
    expose I_Currency as Currency;
    expose I_Country as country;
    }    
    ```
3. Save and activate
### Create service binding
4. Create ZUI_RAP_TRAVEL_O2_SFMX by right clicking on service definition
5. 