[Home - AD260](../../README.md)

# Exercise 5: Use the Background Processing Framework (bgPF) for Asynchronous Calls

## Introduction

In the previous exercise, you've implemented event-driven side effects and simulated an external EML call triggering this side-effect (_see [Exercise 4](../ex04/README.md)_). 

In this exercise, you will learn how to use the Background Processing Framework (bgPF) to asynchronously trigger an external process from within a RAP BO. You will be calling the dummy review workflow process implemented in the previous exercise in the method `REVIEW_TRAVEL_BO()` of the ABAP class ![](../images/adt_class.png)`ZRAP201_START_BGPF_###`.

To achieve that, you will adapt the logic of the action `bookTravel` so that it sets the travel status of the selected _travel_ instances to _Planned_ (`P`) and triggers the review workflow. The workflow will either set the travel status to _Booked_ (`B`) or _Cancelled_ (`X`) depending on the result. 
  
## Exercises

- [5.1 - Adjust the _Travel_ BO Behavior Pool - Part 1](#exercise-51-adjust-the-travel-bo-behavior-pool---part-1)
- [5.2 - Create the bgPF Starter Class](#exercise-52-create-the-bgpf-starter-class)
- [5.3 - Adjust the _Travel_ BO Behavior Pool - Part 2](#exercise-53-adjust-the-travel-bo-behavior-pool---part-2) 
- [5.4 - Preview and Test the Enhanced _Travel_ App](#exercise-54-preview-and-test-the-enhanced-travel-app)
- [Summary & Next Exercise](#summary--next-exercise)

<br/>

> [!TIP]
> <details>
>  <summary>Click to expand ADT tips!</summary>  
>  
> - Always replace all occurrences of the placeholder **`###`** in the provided code snippets with your personal suffix.
> - Use the ADT function _**Find and Replace All**_ (**Ctrl+F**) to quickly replace text in the source code.
> - Use the ADT function _**Quick Fix**_ (**Ctrl+1**), aka _Quick Assist_, on an erroneous element to get help with resolving the issue.
> - Use the **Show ABAP element info** view (**F2**) to inspect an element in ADT editors.
> - Use the **ABAP Formatter** function (**Ctrl+F1**) to format your source code.
> - [Useful Keyboard Shortcuts for ABAP Development](https://help.sap.com/docs/ABAP_PLATFORM_NEW/c238d694b825421f940829321ffa326a/4ec299d16e391014adc9fffe4e204223.html?version=latest) (ADT shortcuts)
>
> </details>

> [!NOTE]
> **About the Background Processing Framework (bgPF)**
> 
> <details>
>  <summary>Click to expand!</summary>  
>  <br/>
>   
> The **background processing framework (bgPF)** is a framework that asynchronously and reliably executes methods of applications that develop background processes.  
>   
> Technically, **bgPF** builds on the background remote function call (bgRFC), wrapping it and extending its functionality. As a successor of the transactional remote function call (tRFC) and the queued remote function call (qRFC), **bgRFC** delivers improved runtime performance through efficient, scalable, and transactional processing of large numbers of sequential function calls. 
>   
> Unlike bgRFC, which supports inbound, outbound, and out-in scenarios, **bgPF** supports only the inbound scenario. This means the sending system and the receiving system are identical, and the remote call runs in a new session. 
>  
> **Learn more:** [Background Processing Framework in ABAP Cloud](https://help.sap.com/docs/abap-cloud/abap-concepts/background-processing-framework) 
>  </details>

---

## Exercise 5.1: Adjust the _Travel_ BO Behavior Pool - Part 1

> Declare a public type `t_travel_for_change` in the global class of the _Travel_ BO behavior pool ![](../images/adt_class.png)`ZBP_R_TRAVELTP###`.
> 
> This type will be used in the following exercise to easily access the derived BO type outside the _Travel_ RAP BO.

<details>
  <summary>🔵 Click to expand!</summary>

1. Open the _travel_ BO behavior pool ![](../images/adt_class.png)**`ZBP_R_TRAVELTP###`** in the **Project Explorer**.

2. Go to the **Global Class** tab and declare the new type **`t_travel_for_change`** in the public section:

   ```abap
           "to be used in BGPF class
           TYPES t_travel_for_change TYPE STRUCTURE FOR CHANGE zr_traveltp###\\TravelTP.    
   ``` 

   <details>
     <summary>ℹ️📄Click to review the updated source code of the global class!</summary>
     
   ```abap
   CLASS zbp_r_traveltp### DEFINITION
     PUBLIC
     ABSTRACT
     FINAL
     FOR BEHAVIOR OF zr_traveltp### .
    
     PUBLIC SECTION.
       "to be used in BGPF class
       TYPES t_travel_for_change TYPE STRUCTURE FOR CHANGE zr_traveltp###\\TravelTP.
    
     PROTECTED SECTION.
     PRIVATE SECTION.
   ENDCLASS.
           
   CLASS zbp_r_traveltp### IMPLEMENTATION.
   ENDCLASS.
   ```           
   </details>  

3. Save ![save icon](../images/adt_save.png) (**Ctrl+S**) and activate ![activate icon](../images/adt_activate.png) (**Ctrl+F3**) the changes.

</details>


## Exercise 5.2: Create the bgPF Starter Class

> Create and implement the ABAP class ![](../images/adt_class.png)`ZRAP201_START_BGPF_###`, which will be used to start the review process asynchronously later in the _Travel_ BO behavior pool.

<details>
  <summary>🔵 Click to expand!</summary>

1. Right-click on your exercise package ![](../images/adt_package.png)**`ZRAP201_###`** and select **New** > **ABAP Class**.

2. Enter the following values:

   | Field | Value |
   |---|---|
   | Name | **`ZRAP201_START_BGPF_###`** |
   | Description | **`bgPF review workflow starter`** |

3. Replace the entire source code of ![](../images/adt_class.png)**`ZRAP201_START_BGPF_###`** with the version provided below and replace all (**Ctrl+F**) occurrences of the placeholder **`###`** with your personal suffix.

   <!-- ▶🟡📄Source code document: [![](../images/adt_class.png)ex05_class_ZRAP201_START_BGPF_###.txt](sources/ex05_class_ZRAP201_START_BGPF.txt) -->
   
   <details>
     <summary>🟡📄 Click to expand the source code!</summary>

   ```abap
   CLASS zrap201_start_bgpf_### DEFINITION
     PUBLIC
     FINAL
     CREATE PUBLIC .
  
     PUBLIC SECTION.
       INTERFACES if_serializable_object.
       INTERFACES if_bgmc_operation.
       INTERFACES if_bgmc_op_single_tx_uncontr.
  
       CLASS-METHODS run_via_bgpf_tx_uncontrolled
         IMPORTING i_rap_bo_entity                 TYPE zbp_r_traveltp###=>t_travel_for_change
         RETURNING VALUE(r_process_monitor_string) TYPE string.
  
       METHODS constructor
         IMPORTING i_rap_bo_entity TYPE zbp_r_traveltp###=>t_travel_for_change.
  
       CONSTANTS:
         BEGIN OF bgpf_state,
           unknown         TYPE int1 VALUE IS INITIAL,
           erroneous       TYPE int1 VALUE 1,
           new             TYPE int1 VALUE 2,
           running         TYPE int1 VALUE 3,
           successful      TYPE int1 VALUE 4,
           started_from_bo TYPE int1 VALUE 99,
         END OF bgpf_state.
  
     PROTECTED SECTION.
     PRIVATE SECTION.
       DATA rap_bo_line TYPE zbp_r_traveltp###=>t_travel_for_change .
   ENDCLASS.
  
   CLASS zrap201_start_bgpf_### IMPLEMENTATION.
  
     METHOD constructor.
       rap_bo_line = i_rap_bo_entity.
     ENDMETHOD.
  
     METHOD if_bgmc_op_single_tx_uncontr~execute.
  
       DATA rap_update TYPE REF TO zrap201_external_eml_call_### .
  
       rap_update = NEW #(
          i_rap_bo_key = rap_bo_line-uuid
          i_travel_id  = rap_bo_line-TravelID
       ).
  
       rap_update->review_travel_bo( ).
  
     ENDMETHOD.
  
     METHOD run_via_bgpf_tx_uncontrolled.
       TRY.
           DATA(process_monitor) = cl_bgmc_process_factory=>get_default( )->create(
                                                 )->set_name( |Long running process { i_rap_bo_entity-UUID }|
                                                 )->set_operation_tx_uncontrolled(
                                                      NEW zrap201_start_bgpf_###( i_rap_bo_entity = i_rap_bo_entity )
                                                 )->save_for_execution( ).
           r_process_monitor_string = process_monitor->to_string( ).
         CATCH cx_bgmc INTO DATA(lx_bgmc).
       ENDTRY.
     ENDMETHOD.
  
   ENDCLASS.
   ```   
   </details>     
   
   <details>
     <summary>ℹ️ Brief explanation - Click to expand !</summary>
      
     > - Following Interfaces must be implemented: **`if_serializable_object`**, **`if_bgmc_operation`**, and **`if_bgmc_op_single_tx_uncontr`**. Here is a short explanation of the mentioned interfaces and why they are needed:
     > 
     > - **`if_serializable_object`**
     > This is a marker interface (no methods) that tells the ABAP runtime: "instances of this class can be serialized — converted to a byte stream and reconstructed later."
     > Why bgPF needs it: Background jobs don't run immediately in the caller's memory context. The framework needs to persist the operation object - including its input parameters - to the database, hand it off to a work process (possibly on a different application server), and reconstruct it there before execution. Serialization is the mechanism that makes this handover possible. Without it, the framework cannot save and restore the object's state across process boundaries.
     >
     > - **`if_bgmc_operation`**
     > This is the base interface for all BGPF operations. It defines the contract that the framework uses to interact with your job implementation - for example, how the framework identifies and manages the operation.
     > Why bgPF needs it: It establishes your class as a valid BGPF operation that the framework can discover, queue, and execute. Think of it as the "entry ticket" - without implementing this interface, the framework simply doesn't recognize your class as something it can process.
     >
     > - **`if_bgmc_op_single_tx_uncontr`**
     > This interface defines the execution mode: single transaction, uncontrolled. It provides the EXECUTE method where you put your actual business logic.
     > Single transaction means your operation runs in one database transaction - either everything commits, or everything rolls back. Uncontrolled means the framework does not inject any transaction control around your EXECUTE method call - you are responsible for COMMIT WORK if needed, giving you full control over the transaction boundary.
     > Why bgPF needs it: BGPF supports different execution modes (e.g. controlled vs. uncontrolled transaction handling). By choosing this specific interface, you explicitly declare how your operation participates in transaction management. This is important in ABAP Cloud because implicit COMMIT WORK calls (as in classic ABAP) are not allowed - transaction boundaries must be explicit and intentional. The framework uses the interface to know which execution protocol to apply when running your operation.
     > 
     > - **`constructor()`**: Hand-over of the given _Travel_ BO entity. The constructor is called when the bgPF instance is being prepared. By this, the travel BO key is handed over into the bgPF and serialized accordingly. When the bgPF instance is executed, the key is de-serialized and handed over to the execute method for the asynchronous processing.
     >  
     > - **`run_via_bgpf_tx_uncontrolled()`**: main method implemented as class method. This is the method called from the save_modified implementation in the RAP BO. Inside its implementation the **`cl_bgmc_process_factory`** is used to build and execute the bgPF instance.
       
   </details>  

3. Save ![save icon](../images/adt_save.png) (**Ctrl+S**) and activate ![activate icon](../images/adt_activate.png) (**Ctrl+F3**) the changes.

   <br/>
   
</details>

## Exercise 5.3: Adjust the _Travel_ BO Behavior Pool - Part 2

> Adjust the business logic of the _Travel_ BO behavior Pool ![](../images/adt_class.png)`ZBP_R_TRAVELTP###`. First, you will adjust the business logic of the implementation of the action `bookTravel()` in the local handler class `lhc_ZR_TravelTP###` to reflect the new process, then you will enhance the business logic of the `save_modified()` method of the local saver class `lsc_ZR_TravelTP###` to trigger the asynchronous review workflow.

<details>
  <summary>🔵 Click to expand!</summary>

### Exercise 5.3.1: Adjust the Implementation of the `bookTravel()` Method in Local Handler Class
[^Top of page](#)

> Adjust the business logic of the implementation of the action `bookTravel()` in the local handler class **`lhc_ZR_TravelTP###`** of the _Travel_ BO behavior Pool.
>  
> From now on, when the `bookTravel` action is triggered, the travel status of the selected _travel_ instances is set to _Planned_ (`P`), and the review workflow is triggered and processed asynchronously. The workflow then sets the travel status to _Booked_ (`B`) or _Cancelled_ (`X`) depending on the processing result. 

<details>
  <summary>🟣 Click to expand!</summary>

1. In ADT, 
   - open the _travel_ BO behavior pool ![](../images/adt_class.png)**`ZBP_R_TRAVELTP###`** in the **Project Explorer**,
   - go to the **Local Types** tab, and
   - navigate to the implementation of the method **`bookTravel()`** in the local handler class **`lhc_ZR_TravelTP###`**.

2. Now, set the fields travel status **`Status`** and the review status **`ReviewStatus`** to planned, i.e. `P` and `2` respectively, and change the notification **`Notification`** to **`Travel set to planned and sent to review`** in the **`MODIFY ENTITIES`** statement: 

   > - `Status`       = zrap201_if_traveltp###=>travel_status-planned    "`P` 
   > - `ReviewStatus` = zrap201_if_traveltp###=>review_status-planned    "`2` 
   > - `Notification` = 'Travel set to planned and sent to review'

   To achieve this,
   - go to the implementation of the action method **`bookTravel`**, and
   - replace the **`MODIFY ENTITIES`** statement with the version provided below. 
   
     Remember to replace the placeholder **`###`** with your personal suffix.
     
     ```abap
         MODIFY ENTITIES OF ZR_TravelTP### IN LOCAL MODE
            ENTITY TravelTP
            UPDATE FIELDS ( Status ReviewStatus Notification )
            WITH VALUE #( FOR key IN keys ( %tky    = key-%tky
                                             Status       = zrap201_if_traveltp###=>travel_status-planned  "ex05
                                             ReviewStatus = zrap201_if_traveltp###=>review_status-planned  "ex05
                                             Notification = 'Travel set to planned and sent to review'     "ex05
                                           ) )
         FAILED failed
         REPORTED reported.
     ```           

   The updated implementation of the **`bookTravel()`** method now looks as follows: 

   <details>
     <summary>ℹ️📄 Click to expand the source code!</summary>
    
   ```abap
      METHOD bookTravel.
        "modify travel instance(s)
        "update the travel status `Status`, as well as `ReviewStatus`and `Notification`
        MODIFY ENTITIES OF ZR_TravelTP### IN LOCAL MODE
           ENTITY TravelTP
           UPDATE FIELDS ( Status ReviewStatus Notification )
           WITH VALUE #( FOR key IN keys ( %tky    = key-%tky
                                            Status       = zrap201_if_traveltp###=>travel_status-planned   "ex05
                                            ReviewStatus = zrap201_if_traveltp###=>review_status-planned   "ex05
                                            Notification = 'Travel set to planned and sent to review'      "ex05
                                          ) )
        FAILED failed
        REPORTED reported.
      
        "read changed data for action result
        READ ENTITIES OF ZR_TravelTP### IN LOCAL MODE
           ENTITY TravelTP
           ALL FIELDS WITH
           CORRESPONDING #( keys )
           RESULT DATA(travels).
      
        "set the action result parameter
        result = VALUE #( FOR travel IN travels ( %tky   = travel-%tky
                                                  %param = travel ) ).
      ENDMETHOD.
   ```
   
   </details>  

4. Save ![save icon](../images/adt_save.png) (**Ctrl+S**) the changes and continue with the next step.

   <br/>
   
</details>

### Exercise 5.3.2: Adjust the Implementation of the Local Saver Class

> Adjust the business logic of the implementation of the `save_modified()` method in the local saver class `lsc_zr_traveltp###` of the _Travel_ BO behavior Pool.
> 
> From now on, the review workflow is started asynchronously whenever the travel status `Status` is changed to _Planned_ (`P`), in addition to the `StatusUpdated` event, which is raised every time the travel status changes. 

<details>
  <summary>🟣 Click to expand!</summary>

1. Now, go to the implementation of the method **`save_modified()`** in the local saver class **`lsc_zr_traveltp###`** of the _Travel_ BO behavior pool ![](../images/adt_class.png)**`ZBP_R_TRAVELTP###`**.

2. Replace the implentation of the **`save_modified()`** method. 
   
   The updated implementation of the **`bookTravel()`** method introduces the call to the bgPF framework and looks as follows: 

   <details>
     <summary>ℹ️📄Click to view the source code!</summary>
    
   ```abap
    METHOD save_modified.
      DATA: events_to_be_raised TYPE TABLE FOR EVENT ZR_TravelTP###~StatusUpdated.
  
      "raise the event whenever the status is changed
      IF update-traveltp IS NOT INITIAL.
        LOOP AT update-traveltp INTO DATA(update_traveltp).

          "check if travel status has changed
          IF update_traveltp-%control-Status = if_abap_behv=>mk-on.
  
            "ex05_begin
            "start review workflow only if travel status has been set to planned
            IF update_traveltp-Status = zrap201_if_travelTP###=>travel_status-planned.  "ex05_begin
              "start the asynchronous review workflow via bgPF
              DATA(process_monitor_string) = zrap201_start_bgpf_###=>run_via_bgpf_tx_uncontrolled(
                EXPORTING
                  i_rap_bo_entity          = update_traveltp
              ).
            ELSE.
              "Planned is considered as an intermediate status, so we don't raise the event for it.
              "We only raise the event when the status is changed to Cancelled or New.
              APPEND CORRESPONDING #( update_traveltp ) TO events_to_be_raised.
            ENDIF.
            "ex05_end                                                                   "ex05_end

          ENDIF.
        ENDLOOP.

        IF events_to_be_raised IS NOT INITIAL.
          RAISE ENTITY EVENT ZR_TravelTP###~StatusUpdated FROM events_to_be_raised.
        ENDIF.
      ENDIF.
    ENDMETHOD.
   ```
   </details> 

3. You can take a look at the changed lines (tagged with `//ex05`) in the behavior pool ![](../images/adt_class.png)**`ZBP_R_TRAVELTP###`** in the document linked below:

   > ℹ️📄Source code document: [![](../images/adt_class.png)ex05_class_ZBP_R_TRAVELTP###.txt](sources/ex05_class_ZBP_R_TRAVELTP.txt)

4. Save ![save icon](../images/adt_save.png) (**Ctrl+S**) and activate ![activate icon](../images/adt_activate.png) (**Ctrl+F3**) the changes.

   <br/>
   
</details>

</details>

## Exercise 5.4: Preview and Test the Enhanced _Travel_ App

> Preview and test the enhanced _Travel_ app in your exercise package ![](../images/adt_package.png)`ZRAP201_###`.

<details>
  <summary>🔵 Click to expand!</summary>

1. Refresh the app in the browser, or go to the Service Binding ![](../images/adt_srvb.png)**`ZUI_TRAVELTP_O4###`**, select the **Travel** entity (leading entity) and start the **SAP Fiori Elements App Preview**.

2. Press **Go** to load the data in the app.

3. Select a travel record with the travel status **`N`** (_New_) and press **Book Travel**. 

4. Verify that:
   - The travel status is set to **`P`** (_Planned_) and the notification set to `Travel set to planned and sent to review`.
   - After a few seconds, you should get notified that the data is changed and the travel status will be updated to **`B`** (_Booked_) or **`X`** (_Cancelled_) according to the total price. The notification will be updated accordingly.
     - Travel is _booked_ (`B`) by the review workflow is is higher than `0` but lower than `3.000`
     - Travel is _cancelled_ (`X`) otherwise

   <br/>
   <img src="images/ex05-playaround-with-app.png" alt="Play around with the app and bgPF" width="100%"> 

5. Play around with enhanced app.

</details>


## Summary & Next Exercise
[^Top of page](#introduction)

Now that you've...
- implemented an ABAP class to start business via bgPF,
- adapted the business logic of the `bookTravel` action, and
- start the review process asynchronously using bgPF in the save sequence of the _Travel_ BO runtime,

you can continue with – **[Exercise 6: Work with the Read-Only RAP Analytical Table](../ex06/README.md)**.

---
