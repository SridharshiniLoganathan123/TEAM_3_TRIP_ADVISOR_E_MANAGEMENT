# TripAdvisor-E-Management
Here's a README format based on your requirements:

---

# TripAdvisor Automation in Salesforce

## Overview
This Salesforce solution manages the data and automates key processes for Hotels, Flights, Food Options, and Customer Discounts. The automation simplifies data management and enhances the customer experience through timely notifications and targeted discounts.

### Key Objectives
1. **Hotel Management**: Automatically update hotel information when a new food option is added or updated.
2. **Customer Discounts**: Apply discounts to customer bills when their total purchase amount exceeds a specified threshold.
3. **Flight Reminders**: Schedule reminder emails to be sent 24 hours before a flight departure with confirmation of successful email delivery.

---

## Solution Components

### 1. Custom Objects and Fields
The following custom objects and fields are created to support data management:

- **Hotel**: Stores hotel information and includes fields like `Name`, `Location`, and `TotalFoodOptions__c`.
- **Food Options**: Represents various food options with fields like `Name`, `Hotel__c` (lookup to `Hotel`), and `Price__c`.
- **Customer**: Contains customer details with fields like `Name`, `Email`, and `TotalPurchaseAmount__c`.
- **Flights**: Stores flight booking details with fields such as `Name`, `DepartureDateTime__c`, and `ContactEmail__c`.

---

### 2. Automation

#### Apex Trigger on Food Options
An Apex trigger ensures that whenever a new food option is added or updated, the corresponding hotel’s total food option count is updated.

**Trigger Code**:
```apex
trigger FoodOptionTrigger on Food_Option__c (after insert, after update, after delete) {
    if (trigger.isInsert || trigger.isUpdate) {
        FoodOptionTriggerHandler.updateHotelFoodOptionCount(trigger.new);
    } else if (trigger.isDelete) {
        FoodOptionTriggerHandler.updateHotelFoodOptionCount(trigger.old);
    }
}
```

**Handler Class**:
```apex
public class FoodOptionTriggerHandler {
    public static void updateHotelFoodOptionCount(List<Food_Option__c> foodOptions) {
        Set<Id> hotelIds = new Set<Id>();
        for (Food_Option__c option : foodOptions) {
            hotelIds.add(option.Hotel__c);
        }

        List<Hotel__c> hotels = [SELECT Id, TotalFoodOptions__c, (SELECT Id FROM Food_Options__r) 
                                  FROM Hotel__c 
                                  WHERE Id IN :hotelIds];

        for (Hotel__c hotel : hotels) {
            hotel.TotalFoodOptions__c = hotel.Food_Options__r.size();
        }
        update hotels;
    }
}
```

---

#### Flow for Customer Discounts
A Record-Triggered Flow applies a discount to a customer’s total bill when their total purchase amount (`TotalPurchaseAmount__c`) reaches a certain threshold.

1. **Trigger**: When a `Customer` record is created or updated.
2. **Condition**: Checks if `TotalPurchaseAmount__c` exceeds the specified discount threshold.
3. **Action**: Updates the `Discount__c` field on the customer’s record.

---

#### Apex Schedulable Class for Flight Reminder Emails
The schedulable class sends reminder emails 24 hours before the scheduled flight departure and confirms successful email delivery.

**Schedulable Class Code**:
```apex
public class FlightReminderScheduledJob implements Schedulable {
    public void execute(SchedulableContext sc) {
        sendFlightReminders();
    }

    public void sendFlightReminders() {
        Datetime now = Datetime.now();
        Datetime tomorrow = now.addDays(1);

        List<Flight__c> upcomingFlights = [
            SELECT Id, Name, DepartureDateTime__c, ContactEmail__c 
            FROM Flight__c 
            WHERE DepartureDateTime__c >= :now 
            AND DepartureDateTime__c <= :tomorrow
        ];

        for (Flight__c flight : upcomingFlights) {
            Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
            email.setToAddresses(new List<String>{ flight.ContactEmail__c });
            email.setSubject('Flight Reminder: ' + flight.Name);
            email.setPlainTextBody('This is a reminder for your upcoming flight ' + flight.Name +
                                   ' departing on ' + flight.DepartureDateTime__c);

            try {
                Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ email });
                System.debug('Confirmation: Reminder email sent successfully for Flight ' + flight.Name);
            } catch (Exception e) {
                System.debug('Error sending email for Flight ' + flight.Name + ': ' + e.getMessage());
            }
        }
    }
}
```

**Scheduling the Job**:
To run this job daily, schedule it with the following code in **Developer Console**:
```apex
String cronExp = '0 0 6 * * ?'; // Runs daily at 6:00 AM
System.schedule('DailyFlightReminder', cronExp, new FlightReminderScheduledJob());
```

---

## Summary
This Salesforce solution leverages custom objects, Apex triggers, Flows, and a schedulable class to automate TripAdvisor’s data management for Hotels, Food Options, Customers, and Flights. By implementing these automations, the system can automatically update hotel data, apply customer discounts, and send timely flight reminders.
