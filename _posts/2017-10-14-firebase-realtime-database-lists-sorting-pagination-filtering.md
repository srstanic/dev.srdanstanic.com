---
layout: post
title:  "How to sort, filter, and paginate lists in the Firebase Realtime Database?"
categories: firebase
redirect_from:
  - /2017/10/14/firebase-realtime-database-lists-sorting-pagination-filtering/
---

Let's say you are building an app for the secret service. One of the requirements is to have a list of agents sorted by the timezone they are currently operating in. Here is how a list of agent objects might be defined in the Firebase Realtime Database:

```json
{
  "agents": {
    "-KwFKidh4OKGlt8nak4A": {
      "name": "Robbert Ridger",
      "timezone_gmt_offset": 0,
      "timezone_id": "Africa/Abidjan"
    },
    "-KwFKidms27MFp6xZ6HV": {
      "name": "Tiphany Broader",
      "timezone_gmt_offset": 1,
      "timezone_id": "Africa/Lagos"
    },
    "-KwFKidnpXvgxrVYgO9z": {
      "name": "Eveline Klaffs",
      "timezone_gmt_offset": -3,
      "timezone_id" :"America/Sao_Paulo"
    },
    "-KwFKidp8zEQH7yOT-nI": {
      "name": "Umeko Piche",
      "timezone_gmt_offset": -4,
      "timezone_id" :"America/Toronto"
    },
    ...
  }
}
```

Each agent object has a property named `timezone_gmt_offset` with a signed number value representing the GMT offset.

Out of the box, Firebase Realtime Database offers a couple of ways to sort and paginate object lists. These [two](https://howtofirebase.com/collection-queries-with-firebase-b95a0193745d){:target="_blank"}<!-- markup clean_ --> [articles](https://howtofirebase.com/firebase-data-structures-pagination-96c16ffdb5ca){:target="_blank"}<!-- markup clean_ --> cover the basics of querying and pagination pretty well. I suggest you go read them if you are just starting out with the Firebase Realtime Database and then come back.

Pagination
----------

Now, let's assume that for the efficiency reasons we decided to implement the cursor based pagination. Here is how we will go about implementing it.

We will use the `orderByChild()` method to specify the sorting and the `timezone_gmt_offset` property to sort the agents by the timezone they are operating in. To limit the results to the desired page size, we will use the `limitToFirst()` method. When retrieving data for one page, we will get N+1 items, where N is the size of the page. That additional item we retrieve is used as a cursor for the next page. In our examples, we will make the page size to be 4 and for each page we will retrieve 5 items:

```javascript
firebase.database()
        .ref("/agents")
        .orderByChild("timezone_gmt_offset")
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```

A resulting snapshot might look like the following JSON snippet:

```json
{
    "-KwFKidp8zEQH7yOT-nI": {
        "name": "Umeko Piche",
        "timezone_gmt_offset": -4,
        ...
    },
    "-KwFKidnpXvgxrVYgO9z": {
        "name": "Eveline Klaffs",
        "timezone_gmt_offset": -3,
        ...
    },
    "-KwFKidh4OKGlt8nak4A": {
        "name": "Robbert Ridger",
        "timezone_gmt_offset": 0,
        ...
    },
    "-KwFKidms27MFp6xZ6HV": {
        "name": "Tiphany Broader",
        "timezone_gmt_offset": 1,
        ...
    },
    "-KwFKidwW6ZCE7Yebfpj": {
        "name": "Tabbie Borg",
        "timezone_gmt_offset": 2,
        ...
    }
}
```

Notice that the items are sorted numerically from the smallest to the largest GMT offset. When handling the snapshot though, you need to be aware that you have received a JSON object and if you try to iterate over its keys, they will not be sorted as you would expect. Here is the code that will maintain the proper sorting:

```javascript
snapshot.forEach(function(child) {
    var agent_id = child.key
    var agent = child.val()
    // do what you need to do with the agent object, e.g. populate the list view item
});
```

After getting the results, we cut off the last item and present the N items in the view, which is 4 in our example.

```
Umeko Piche (GMT-4)
Eveline Klaffs (GMT-3)
Robbert Ridger (GMT+0)
Tiphany Broader (GMT+1)
```

When the user initiates loading of another page, we will query for another N+1 items starting with the last item from the previous page. We will use the `startAt()` method in a combination with the `limitToFirst()` method to define the items range for the second page. The limit of the Firebase Database here is that for the `startAt()` method you can only use the value of the property specified in the `orderByChild()` method.

The problem we face now is that the value of the `timezone_gmt_offset` property is not unique across the agents. And we can't use a value shared by multiple items to specify the starting item of the next page.

To address this issue we are going to create a new property with a unique value for each item. We will name it `_sort_timezone_gmt_offset` and its value will be derived from the `timezone_gmt_offset` property value and the object key:

```javascript
var refPath = "/agents"
firebase.database()
        .ref(refPath)
        .once("value")
        .then(function(snapshot) {
            var updates = {}
            snapshot.forEach(function(child) {
                var agent = child.val()
                updates["/" + child.key + "/_sort_timezone_gmt_offset"] =
                    agent.timezone_gmt_offset + "_" + child.key
            });
            firebase.database().ref(refPath).update(updates);
        })
```

Now we can use the newly added property to order agents by.

```javascript
firebase.database()
        .ref("/agents")
        .orderByChild("_sort_timezone_gmt_offset")
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```

And here are the results for the given query:

```json
{
    "-KwFKidnpXvgxrVYgO9z": {
        "_sort_timezone_gmt_offset": "-3_-KwFKidnpXvgxrVYgO9z",
        "name": "Eveline Klaffs",
        "timezone_gmt_offset": -3,
        ...
    },
    "-KwFKidp8zEQH7yOT-nI": {
        "_sort_timezone_gmt_offset": "-4_-KwFKidp8zEQH7yOT-nI",
        "name": "Umeko Piche",
        "timezone_gmt_offset": -4,
        ...
    },
    "-KwFKidh4OKGlt8nak4A": {
        "_sort_timezone_gmt_offset": "0_-KwFKidh4OKGlt8nak4A",
        "name": "Robbert Ridger",
        "timezone_gmt_offset": 0,
        ...
    },
    "-KwFKiduM8A7hD-xAeyW": {
        "_sort_timezone_gmt_offset": "10_-KwFKiduM8A7hD-xAeyW",
        "name": "Johnathan Mandry",
        "timezone_gmt_offset": 10,
        ...
    },
    "-KwFKie-IJCovyOTklNg": {
        "_sort_timezone_gmt_offset": "12_-KwFKie-IJCovyOTklNg",
        "name": "Finn Kleiser",
        "timezone_gmt_offset": 12,
        ...
    }
}
```

It looks like there is something wrong with the sorting. An agent who is 3 hours behind the GMT comes before an agent who is 4 hours behind the GMT. Agents who are 10 and 12 hours ahead of GMT come before an agent who is only 1 hour ahead. That doesn't make sense. Or does it?

The value of the property by which the items are sorted is not a number anymore but a string. String values are sorted lexicographically. That's why "-3" comes before "-4" and "10" comes before "1_".

To fix the sorting, we need to figure out how to populate the `_sort_timezone_gmt_offset` property so that the values can be sorted lexicographically but follow the order of the numeric GMT offset values.

Since the GMT offset values range from [-12 to 14](https://en.wikipedia.org/wiki/List_of_UTC_time_offsets){:target="_blank"}<!-- markup clean_ -->, we can transform each offset value into a character and use it to populate the `_sort_timezone_gmt_offset` property:

```javascript
function getCharTimezoneForGmtOffset(timezoneGmtOffset) {
    var shiftedOffset = timezoneGmtOffset + 12
    return String.fromCharCode(65 + shiftedOffset)
}

var refPath = "/agents"
firebase.database()
        .ref(refPath)
        .once("value")
        .then(function(snapshot) {
            var updates = {}
            snapshot.forEach(function(child) {
                var agent = child.val()
                updates["/" + child.key + "/_sort_timezone_gmt_offset"] =
                    getCharTimezoneForGmtOffset(agent.timezone_gmt_offset) + "_" + child.key
            });
            firebase.database().ref(refPath).update(updates);
        })
```

Now we can repeat the previous query

```javascript
firebase.database()
        .ref("/agents")
        .orderByChild("_sort_timezone_gmt_offset")
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```

And we will get expected results

```json
{
    "-KwFKidp8zEQH7yOT-nI": {
        "_sort_timezone_gmt_offset": "I_-KwFKidp8zEQH7yOT-nI",
        "name": "Umeko Piche",
        "timezone_gmt_offset": -4,
        ...
    },
    "-KwFKidnpXvgxrVYgO9z": {
        "_sort_timezone_gmt_offset": "J_-KwFKidnpXvgxrVYgO9z",
        "name": "Eveline Klaffs",
        "timezone_gmt_offset": -3,
        ...
    },
    "-KwFKidh4OKGlt8nak4A": {
        "_sort_timezone_gmt_offset": "M_-KwFKidh4OKGlt8nak4A",
        "name": "Robbert Ridger",
        "timezone_gmt_offset": 0,
        ...
    },
    "-KwFKidms27MFp6xZ6HV": {
        "_sort_timezone_gmt_offset": "N_-KwFKidms27MFp6xZ6HV",
        "name": "Tiphany Broader",
        "timezone_gmt_offset": 1,
        ...
    },
    "-KwFKidwW6ZCE7Yebfpj": {
        "_sort_timezone_gmt_offset": "O_-KwFKidwW6ZCE7Yebfpj",
        "name": "Tabbie Borg",
        "timezone_gmt_offset": 2,
        ...
    }
}
```

As we've done before, we cut off the last item and present the first 4 items.

```
Umeko Piche (GMT-4)
Eveline Klaffs (GMT-3)
Robbert Ridger (GMT+0)
Tiphany Broader (GMT+1)
```

When the user initiates the loading of the next page, we use the `_sort_timezone_gmt_offset` property value of the 5th item from the results of the previous query. We pass it as an argument to `startAt()` to specify the first item of the next page.


```javascript
firebase.database()
        .ref("/agents")
        .orderByChild("_sort_timezone_gmt_offset")
        .limitToFirst(5)
        .startAt("O_-KwFKidwW6ZCE7Yebfpj")
        .once("value")
        .then(function(snapshot){ ... })
```

And we recieve the next 5 items

```json
{
    "-KwFKidwW6ZCE7Yebfpj": {
        "_sort_timezone_gmt_offset": "O_-KwFKidwW6ZCE7Yebfpj",
        "name": "Tabbie Borg",
        "timezone_gmt_offset": 2,
        ...
    },
    "-KwFKidzCclEjyI0PHPe": {
        "_sort_timezone_gmt_offset": "O_-KwFKidzCclEjyI0PHPe",
        "name": "Pietra Mathiassen",
        "timezone_gmt_offset": 2,
        ...
    },
    "-KwFKidy9F4Kh4Wc3Zr9": {
        "_sort_timezone_gmt_offset": "P_-KwFKidy9F4Kh4Wc3Zr9",
        "name": "Beatriz Martynka",
        "timezone_gmt_offset": 3,
        ...
    },
    "-KwFKidtvrB7_hdcKDrw": {
        "_sort_timezone_gmt_offset": "R_-KwFKidtvrB7_hdcKDrw",
        "name": "Antonina Swinley",
        "timezone_gmt_offset": 5,
        ...
    },
    "-KwFKidqxtQfK1llcgSY": {
        "_sort_timezone_gmt_offset": "S_-KwFKidqxtQfK1llcgSY",
        "name": "Grady Spencley",
        "timezone_gmt_offset": 6,
        ...
    }
}
```

After cutting off the last item, we present the 4 items on the next page:

```
Tabbie Borg (GMT+2)
Pietra Mathiassen (GMT+2)
Beatriz Martynka (GMT+3)
Antonina Swinley (GMT+5)
```

As the user scrolls through the list, we repeat the process until the number of received items is smaller than the number of items requested, which is 5 in our example. At that point, we have reached the last page.


Filtering
---------

Let's also say that the requirements state that the user can search for the agents by their name.

For the advanced search capabilities, we would need to use a third party service with a full-text search engine. But assuming the requirements will be satisfied with a search that supports only "starts-with" strategy and that the search results do not need to be sorted by the timezone GMT offset, we can fulfill the requirements by using only the Firebase Realtime Database querying API.

We can filter the list with `startAt()` and `endAt()` methods which affect the results depending on the property used in the `orderByChild()` method. Since the agents are filtered by their name, we need to use the value of the `name` property to order by instead of the `timezone_gmt_offset` property. As a result, the list will be filtered and sorted by the `name` property.

So how do we filter the results by the entered query string?

We look for the items in the list that are:
* greater than or equal to the query string
* AND
* less than or equal to the query string appended with a Unicode character that is very high when ordered lexicographically.
I found this technique on [StackOverflow](https://stackoverflow.com/a/40633692/517865){:target="_blank"}<!-- markup clean_ --> and here is how to use it:

```javascript
var queryText = "T"
firebase.database()
        .ref("/agents")
        .orderByChild("_sort_name")
        .startAt(queryText)
        .endAt(queryText + "\uf8ff")
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```

The results of this query are:

```json
{
    "-KwFKidwW6ZCE7Yebfpj":
    {
        "_sort_name": "Tabbie Borg_-KwFKidwW6ZCE7Yebfpj",
        "name": "Tabbie Borg",
        "timezone_gmt_offset": 2,
        ...
    },
    "-KwFKidms27MFp6xZ6HV":
    {
        "_sort_name": "Tiphany Broader_-KwFKidms27MFp6xZ6HV",
        "name": "Tiphany Broader",
        "timezone_gmt_offset": 1,
        ...
    }
}
```

Notice that we didn't use the `name` property to order by. Instead we introduced another property named `_sort_name` which is derived from the `name` property value and the object key, similarly to the `_sort_timezone_gmt_offset` property.


```javascript
var refPath = "/agents"
firebase.database()
        .ref(refPath)
        .once("value")
        .then(function(snapshot) {
            var updates = {}
            snapshot.forEach(function(child) {
                var agent = child.val()
                updates["/" + child.key + "/_sort_name"] =
                    agent.name + "_" + child.key
            });
            firebase.database().ref(refPath).update(updates);
        })
```

And the reason is the same as before. To be able to paginate the filtered results, the values of the property you order by must be unique. And the `name` property is not guaranteed to be unique.

In our example, there aren't enough results to show the second page, but if there were, retrieving the next page would require a little change in the code. Instead of passing `queryText` to the `startAt()` method, we would pass the value of the `_sort_name` property of the item cut off from the previous page. That item is the first item on the next page.

```javascript
var queryText = "T"
firebase.database()
        .ref("/agents")
        .orderByChild("_sort_name")
        .startAt(cutOffItem._sort_name)
        .endAt(queryText + "\uf8ff")
        .limitToFirst(5)
        .once("value")
        .then(function(snapshot){ ... })
```


Summary
-------

If you want to have a cursor based pagination in a list sorted by the value of one of the object properties, you will probably need to create an additional property in the object. The value of that property should be derived from the value of the property you want to sort by and the object key. The reason is that the property by which you sort the paginated list needs to have a unique value.

If you want to filter the object list with a query string matching the value of one of the object properties, it is possible, but only with a "starts-with" approach. When filtering the object list, the ordering must be done by the same property as filtering is done by.


_Thanks to [Miran Brajša](https://www.toptal.com/resume/miran-brajsa){:target="_blank"}<!-- markup clean_ --> for reviewing this post._


References
----------
* [https://firebase.google.com/docs/database/](https://firebase.google.com/docs/database/){:target="_blank"}
* [https://howtofirebase.com/collection-queries-with-firebase-b95a0193745d](https://howtofirebase.com/collection-queries-with-firebase-b95a0193745d){:target="_blank"}
* [https://howtofirebase.com/firebase-data-structures-pagination-96c16ffdb5ca](https://howtofirebase.com/firebase-data-structures-pagination-96c16ffdb5ca){:target="_blank"}
