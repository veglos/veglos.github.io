---
layout: post
title:  "Simple SOLID"
categories:  example
tags: clean solid
---

## Table of contents

1. [Single-Responsibility Principle](#single-responsibility-principle)
2. [Open-Closed Principle](#open-closed-principle)
3. [Liskov Substitution Principle](#liskov-substitution-principle)
4. [Interface Segregation Principle](#interface-segregation-principle)
5. [Dependency Inversion Principle](#dependency-inversion-principle)

---
## 1. Single-Responsibility Principle <a name="single-responsibility-principle"></a>

> There should never be more than one reason for a class to change.

### Violation
```cs
/* This class has two reasons to change: it's properties and storing data in the database */
public class Customer {

    public string Name {get;set}
    public string LastName {get;set}
    public string Email {get;set}
    
    private readonly string _repository;

    public Customer(string repository, string name, string lastName, string email) {
        _repository = repository;
        Name = name;
        LastName = lastName;
        Email = email;
    }

    public void Save() {
                
        if(_repository == "SQLServer") {
            /* Saving in SQLServer*/    
        }
        else if(_repository == "MongoDB") {
            /* Saving in MongoDB*/
        }
        else{
            throw new Exception("Database not supported");
        }
    }
}
```
### Compliance
```cs
/* This class now cares only about it's properties*/
public class Customer {

    public string Name {get;set}
    public string LastName {get;set}
    public string Email {get;set}

    public Customer(string name, string lastName, string email) {
        Name = name;
        LastName = lastName;
        Email = email;
    }
}

/* this class only cares about storing data in the database*/
public class CustomerRepository {

    private readonly string _repository;

    public CustomerRepository(string repository){
        _repository = repository;
    }

    public void Save(Customer customer){
        
        if(_repository == "SQLServer"){
            /* Saving in SQLServer*/
        }
        else if(_repository == "MongoDB"){
            /* Saving in MongoDB*/
        }
        else{
            throw new Exception("Database not supported");
        }
    }
}
```
---

## 2. Open-Closed Principle <a name="open-closed-principle"></a>

> Software entities (classes, modules, functions, etc.)
should be open for extension, but closed for
modification. 

### Violation
```cs
public class EmailNotifier {
    public void Notify(string message){
        /* Notify via e-mail */
    }
}

public class SMSNotifier  {
    public void Notify(string message){
        /* Notify via SMS */
    }
}

public class AlertManager {

    public void SendNotification(List<string> notifiers, string message){
        /* WARNING: New notifiers cannot be added without modifying this module */
        foreach(var notifier in notifiers){
            switch(notifier){
                case "Email": 
                    var email = new EmailNotifier();
                    email.Notify(message);
                case "SMS": 
                    var sms = new SMSNotifier();
                    sms.Notify(message); 
            }
        }
    }
}
```

### Compliance
```cs
public class Notifier {
    public void Notify(string message){
        return;
    }
}

public class EmailNotifier : Notifier {
    public void Notify(string message){
        /* Notify via e-mail */
    }
}

public class SMSNotifier : Notifier {
    public void Notify(string message){
        /* Notify via SMS */
    }
}

public class WhatsAppNotifier : Notifier {
    public void Notify(string message){
        /* Notify via WhatsApp */
    }
}

public class AlertManager {

    public void SendNotification(List<Notifier> notifiers, string message){
        /* New notifiers can be added without modifying this module */
        foreach(var notifier in notifiers){
            notifier.Notify(message);
        }
    }
}
```
---

## 3. Liskov Substitution Principle <a name="liskov-substitution-principle"></a>

> Functions that use pointers or references to base
classes must be able to use objects of derived classes
without knowing it.

### Violation
```cs
public interface IDatabaseRepository {
    bool Connect(string connectionString);
    Table GetTable(string tableName);
}

public class OracleRepository : IDatabaseRepository{

    public bool Connect(string connectionString){
        /* Connect to the database */
    }

    public Table GetTable(string tableName){
        /* fetch table and return it*/
    }
}

public class SQLServerRepository : IDatabaseRepository{

    public bool Connect(string connectionString){
        /* Connect to the database */
    }

    public Table GetTable(string tableName){
        /* fetch table and return it*/
    }
}

/* Imagine MongoDB support was added later in the application: */
public class MongoDBRepository : IDatabaseRepository {
    
    public bool Connect(string connectionString){
        /* Connect to the database */
    }

    public Table GetTable(string tableName){
        /* MongoDB does not have Tables!! */
        throw new NotImplementedException();
    }
}

public class Client {
        
    public void GetTable(IDatabaseRepository repository, string tableName){   
        /* WARNING: If the MongoDBRepository object is passed as a parameter, calling GetTable() will break the application.
        And we cannot check here if the type is either OracleRepository or SQLServerRepository because that would violate the Open-Closed Principle */      
        repository.GetTable(tableName);
    }
}
```
### Compliance
```cs
public interface IDatabaseRepository {
    bool Connect(string connectionString);    
}

public interface IRelationalDatabaseRepository : IDatabaseRepository {
    Table GetTable(string connectionString);    
}

public class OracleRepository : IRelationalDatabaseRepository{

    public bool Connect(string connectionString){
        /* Connect to the database */
    }

    public Table GetTable(string tableName){
        /* fetch table and return it*/
    }
}

public class SQLServerRepository : IRelationalDatabaseRepository{

    public bool Connect(string connectionString){
        /* Connect to the database */
    }

    public Table GetTable(string tableName){
        /* fetch table and return it*/
    }
}

public class MongoDBRepository : IDatabaseRepository {
    
    public bool Connect(string connectionString){
        /* Connect to the database */
    }
}

public class Client {
        
    public void GetTable(IRelationalDatabaseRepository repository, string tableName){
        /* Now we are safe from exceptions, because every instance of IRelationalDatabaseRepository must handle the GetData method.*/
        repository.GetTable(tableName);
    }
}
```

---
## 4. Interface Segregation Principle <a name="interface-segregation-principle"></a>

> clients should not be forced to depend upon interfaces
that they do not use.

### Violation
```cs
public interface IDatabaseRepository {
    Table GetTable(string tableName);
    /* Imagine MongoDB support was added later in the application: */
    Collection GetCollection(string collectionName);
}

public class SQLServerRepository : IDatabaseRepository {

    public Table GetTable(string tableName) {
        /* fetch table and return it*/
    }


    public Collection GetCollection(string collectionName) {
        /* 
           Method GetCollection is being forced on this class.
           SQLServer does not have Collections! 
        */
        throw new NotImplementedException();
    }
}

/* Imagine MongoDB support was added later in the application: */
public class MongoDBRepository : IDatabaseRepository {

    public Table GetTable(string tableName) {
        /* 
           Method GetTable is being forced on this class.
           MongoDB does not have Tables! 
        */
        throw new NotImplementedException();
    }

    public Collection GetCollection(string collectionName) {
        /* fetch collection and return it*/
    }
}
```

### Compliance
```cs
/* Split the IDatabaseRepository interface between IRelationalDatabaseRepository and INonRelationalDatabaseRepository so no implementation is forced to handle methods it doesn't need.*/
public interface IDatabaseRepository {
    
}

public interface IRelationalDatabaseRepository : IDatabaseRepository {
    Table GetTable(string tableName);
}

public interface INonRelationalDatabaseRepository : IDatabaseRepository {
    Collection GetCollection(string collectionName);
}

public class SQLServerRepository : IRelationalDatabaseRepository {

    public Table GetTable(string tableName) {
        /* fetch table and return it*/
    }
}

public class MongoDBRepository : INonRelationalDatabaseRepository{

    public Collection GetCollection(string collectionName) {
        /* fetch collection and return it*/
    }
}
```

---
## 5. Dependency Inversion Principle <a name="dependency-inversion-principle"></a>

> a. high level modules should not depend upon low
level modules. both should depend upon abstractions.
>
>b. abstractions should not depend upon details. details
should depend upon abstractions.

### Violation
```cs
public class CustomerRepository {

    private readonly string _repository;

    public CustomerRepository(string repository){
        _repository = repository;
    }

    public void Save(Customer customer){
        
        /* WARNING: The high level module CustomerRepository DEPENDS on the low level modules SQLServerRepository */
        if(_repository == "SQLServer"){
            var repository = new SQLServerRepository();
            /* Saving in SQLServer*/
        }
        else{
            throw new Exception("Database not supported");
        }
    }
}
```
### Compliance
```cs

public interface IDatabaseRepository{
    void Save(Customer customer);
}

public class SQLServerRepository : IDatabaseRepository {

    public void Save(Customer customer){
        /* Saving in SQLServer*/
    }
}

public class CustomerRepository {

    private readonly IDatabaseRepository _repository;

    public CustomerRepository(IDatabaseRepository repository){
        _repository = repository;
    }

    /* Now the high level module CustomerRepository and the low level modules SQLServerRepository and MongoDBRepository DEPEND on the abstraction IDatabaseRepository */
    public void Save(Customer customer){        
        _repository.Save(customer);
    }
}
```