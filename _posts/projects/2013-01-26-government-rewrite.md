---
layout: post
title: Government Rewrite
tagline: 
---
{% include JB/setup %}

Follows are a few notes for a system I'm about to begin work on. The idea is to provide a system
that can help improve transparency and efficiency in government systems - in their entirety. 

This is based on thoughts I've been throwing around recently after a few experiences over the year
and with recent news bringing PACER into light. It's both a trivial and tremendous undertaking,
in that the underlying technology would be easy to implement, but adoption would be quite 
difficult. Again, very light on details here, but we need to start somewhere.

Government Rewrite:
===================================

A system for historically versioned, authenticated, schema-aware document management.
-----------------------------------

Basic Goal:

To develop an application development platform upon which a system capable of handling all 
government based bureaucratic systems.

* Legal Documentation
* Identity Management / Verification
* Bid Management
* Campaign Finances
* Police Reports
* Court Case Management
* Work Reports
* Hour Tracking
* Document Management
* Accounting

Technical Design Goal:

1) Develop applications with evolving schemas with minimal developer pain.
2) Access data across multiple applications while maintaining schema variations across these platforms.
3) Maintain access controls on data where absolutely necessary, but indicate presence of that data.

Base Document:

  {
    @: {
      schema: {
        me:
        original:
      }

      version: {
        latest    bool
        parent:
      }

      dates: {
        updated:
        created:
      }

      author: {
        id:
        application:
      }
    }
  }

Reusable Data
--------------------------------------

Data is originally authored by an individual or 
a system acting as an individual. 

Data is authored on an application, which originally
provided the interface for adding the data to the 
system.

Data is operated on and modified by an application,
which may or may not have been the application
the data was originally authored on. 

The authoring application determines if the data 
may be read by another application, and may define a 
set of applications with the ability to read that 
information. This access control may cover an entire
schema, or portions of that schema.

Schema definitions and access controls are public. 
To promote open standards and uniform data access, 
an application should only control read access to 
data when it's critical to the security of the 
application. (eg: Password protection)

```go
  // Schema:
  type Document struct {
    Name:       string,
    Sentences:  []string
  }

  // Access Control:
  AccessControl(Document, func(app) bool {
    return app == "XYZ"
  })
```

```
  // Schema:
  type Member struct {
    Name:       string,
    Password:   []byte
  }

  // Access Control:
  AccessControl(Document.Password, func(app) bool {
    return app == "XYZ"
  })
```

Care should be given to schema migrations in 
this matter, as well. This behavior must be more
thoroughly defined before building based on it.

There are no updates or deletions, and
it is the application's responsibility to filter
this data to the client appropriately. ie: a deleted
comment on a blog post will not actually be removed,
but the application may utilize a schema with an
'approved' boolean value and display only 'approved'
comments to an end user.

```
  type Post struct {
    Active:     bool,
    ...
  }

  client.Search(Post, "Active: true")
```



Versioned Data
--------------------------------------

Data is never removed from the system. If an application
modifies the data, a new record is inserted into the system
and the 'latest' flag is toggled on it. 

Adaptable Schemas
--------------------------------------

Schema Definition:

```
  type Schema struct {
    Id:     UniqueKey
    Parent: UniqueKey
    Up:     func up(source *Record, destination *Record)
    Down:   func down(source *Record, destination *Record)
  }
```

Where up and down are functions that alter
the data from the parent schema and to the target. 

When a schema does not provide a 'down' migration
function, it is considered a 'root.' Roots should 
be avoided, as they heavily prevent systems from
maintaining backwards compatibility

Schema Application Change
-------------------------

Consider an application that presents a calendar to 
a user, and allows them to enter events:

Base => Event
Base => Calendar

Event @ (EventV1): {
  name:
  date:
  calendar: Relation[CalendarV1]
}

Calendar @ (CalendarV1): {
  name:
}

Now, the first version of the application does well,
and realizes an upgrade is necessary:

Event @ (EventV2): {
  name: 
  date:
  time:
  calendar: Relation[CalendarV2]
}

Calendar @ (CalendarV2): {
  name: 
  description:
}

Up and down migrations are created:

  EventV1 => EventV2
  up(source, destination){
    destination.time = MIDNIGHT 
  }

  down(source, destination){
    nil
  }

  CalendarV1 => CalendarV2
  up(source, destination){
    destination.description = ""
  }

  down(source, destination){
    nil
  }


Now, version 1 of the application can still function,
and version 2 of the application can access data 
provided by version 1.

Version 2 of the application may operate on the 
migrated data in one of the following ways:

1)  Continue to operate on sane defaults provided
    by the migrated V1=>V2 data.

    Calendar(version: "V2")

2)  Request only access to new V2 data, ignoring V1

    Calendar(lock: "V2")

3)  Force the upgrade of existing V1 data manually
    to the V2 standard. This operation is not 
    automatic, though utilities may be provided to
    assist with it. 

    Calendar(lock: "V2")

    func migrate(){
      for _, v := range Calendar(version: "V1") {
        v.upgrade(:version => "V2", {
          name:         v.name,
          description:  input
        }
      }
    }

As a side effect of the downward migration path,
version 1 of the application will still run accurately.

    Calendar(base: "V1")

Since the CalendarV2 schema is not a root schema, when
a new CalendarV2 schema post is added to the system,
a CalendarV1 entry will be made for it, as well.
