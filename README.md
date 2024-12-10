```sh
USER
id, name, email, location (lat, lon), notification_preference

TRUCK
(id, truck_id, location (lat, lon), timestamp)
```

```json
{
  _id: ObjectId("areaId"),
  name: "Downtown Area",  // The name of the area
  type: "polygon",  // Can be "polygon" or "circle"
  geometry: {
    type: "Polygon",  // Or "Point" for circle
    coordinates: [
      [
        [-73.935242, 40.730610],
        [-73.935000, 40.730800],
        [-73.934800, 40.730600],
        [-73.935242, 40.730610]
      ]
    ]  // Coordinates of the polygon (if type is "polygon")
  },
  radius: 1000,  // Radius in meters (only if type is "circle")
  trucks_assigned: [ObjectId("truckId1"), ObjectId("truckId2")]  // List of trucks assigned to this area
}
```

> **_The areas collection allows admins to define geographical zones for trucks to operate in. These zones can be polygons (for specific boundaries) or circles (for a given radius around a point)._**

```json
{
  _id: ObjectId("userId"),
  name: "John Doe",
  email: "john.doe@example.com",
  location: {
    type: "Point",  // GeoJSON format for 2dsphere index
    coordinates: [-73.935242, 40.730610]  // [longitude, latitude]
  },
  assigned_area: ObjectId("areaId"),  // The area this user belongs to
  notification_preference: "push"  // Or "email", "sms" based on user preference
}
```

> **_Each user can be assigned to one or more areas. The user's location will be checked to determine if they belong to a defined area._**

```json
{
  _id: ObjectId("truckId"),
  truck_name: "Truck A",
  current_location: {
    type: "Point",  // GeoJSON format for 2dsphere index
    coordinates: [-73.935242, 40.730610]  // [longitude, latitude]
  },
  assigned_areas: [ObjectId("areaId1"), ObjectId("areaId2")],  // List of areas this truck is assigned to
  status: "active",  // Example status (active, inactive)
  last_updated: ISODate("2024-12-10T12:00:00Z")
}
```

> **_The trucks collection will track the dynamic location of the trucks. Trucks will be assigned to one or more areas._**

```json
{
  _id: ObjectId("notificationId"),
  user_id: ObjectId("userId"),
  truck_id: ObjectId("truckId"),
  notification_type: "push",  // Or "email", "sms"
  message: "The waste truck is approaching your location!",
  sent_at: ISODate("2024-12-10T12:05:00Z"),
  status: "sent"  // You can track whether the notification was delivered
}
```

---

```js
app.post("/areas", async (req, res) => {
  const { name, type, geometry, radius } = req.body;
  const area = await db.collection("areas").insertOne({
    name,
    type,
    geometry,
    radius: type === "circle" ? radius : undefined, // Only include radius for circle
  });
  res.json(area.ops[0]);
});
```

```js
app.put("/areas/:areaId/assign-truck/:truckId", async (req, res) => {
  const { areaId, truckId } = req.params;
  await db
    .collection("areas")
    .updateOne(
      { _id: ObjectId(areaId) },
      { $addToSet: { trucks_assigned: ObjectId(truckId) } }
    );
  await db
    .collection("trucks")
    .updateOne(
      { _id: ObjectId(truckId) },
      { $addToSet: { assigned_areas: ObjectId(areaId) } }
    );
  res.json({ message: "Truck assigned to area successfully." });
});
```

```js
app.put("/users/:userId/assign-area/:areaId", async (req, res) => {
  const { userId, areaId } = req.params;
  await db
    .collection("users")
    .updateOne(
      { _id: ObjectId(userId) },
      { $set: { assigned_area: ObjectId(areaId) } }
    );
  res.json({ message: "User assigned to area successfully." });
});
```

```js
async function checkTruckAreaProximity(truckId, truckLongitude, truckLatitude) {
  const truckLocation = {
    type: "Point",
    coordinates: [truckLongitude, truckLatitude],
  };

  const truck = await db
    .collection("trucks")
    .findOne({ _id: ObjectId(truckId) });

  for (let areaId of truck.assigned_areas) {
    const area = await db
      .collection("areas")
      .findOne({ _id: ObjectId(areaId) });

    if (area.type === "polygon") {
      const isInArea = await db
        .collection("users")
        .find({
          location: {
            $geoWithin: { $geometry: area.geometry },
          },
          assigned_area: area._id,
        })
        .toArray();

      isInArea.forEach((user) => {
        sendNotification(user._id, "The waste truck is approaching your area!");
      });
    } else if (area.type === "circle") {
      const distance = calculateDistance(
        truckLongitude,
        truckLatitude,
        area.geometry.coordinates[0],
        area.geometry.coordinates[1]
      );
      if (distance <= area.radius) {
        // Notify users in the area
        const usersInArea = await db
          .collection("users")
          .find({ assigned_area: area._id })
          .toArray();
        usersInArea.forEach((user) => {
          sendNotification(
            user._id,
            "The waste truck is approaching your area!"
          );
        });
      }
    }
  }
}
```

```js
function calculateDistance(lon1, lat1, lon2, lat2) {
  const R = 6371; // Radius of the Earth in km
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLon = ((lon2 - lon1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
      Math.cos((lat2 * Math.PI) / 180) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  const distance = R * c; // Distance in km
  return distance * 1000; // Convert to meters
}
```

```js
async function findNearbyTruck(userId, radiusInMeters) {
  // Get the user's location
  const user = await db.collection("users").findOne({ _id: ObjectId(userId) });

  const userLocation = user.location.coordinates; // [longitude, latitude]

  // Find trucks within the specified radius from the user's location
  const trucks = await db
    .collection("trucks")
    .find({
      current_location: {
        $nearSphere: {
          $geometry: {
            type: "Point",
            coordinates: userLocation, // [longitude, latitude]
          },
          $maxDistance: radiusInMeters, // Distance in meters
        },
      },
    })
    .toArray();

  return trucks;
}
```

```js
async function sendNotification(userId, message) {
  // Here you can implement logic to send notifications (push, email, etc.)
  console.log(`Sending notification to User ${userId}: ${message}`);
}

async function checkAndNotify(userId, radiusInMeters) {
  const trucksNearby = await findNearbyTruck(userId, radiusInMeters);

  if (trucksNearby.length > 0) {
    trucksNearby.forEach((truck) => {
      sendNotification(
        userId,
        `Truck ${truck.truck_name} is approaching your area.`
      );
    });
  }
}
```

```js
async function findTrucksInsideArea(areaId) {
  const area = await db.collection("areas").findOne({ _id: ObjectId(areaId) });

  if (area.type === "circle") {
    const { coordinates, radius } = area.geometry;

    const trucksInArea = await db
      .collection("trucks")
      .find({
        current_location: {
          $geoWithin: {
            $centerSphere: [coordinates, radius / 3963.2], // radius in radians (radius / 3963.2 converts miles to radians)
          },
        },
      })
      .toArray();

    return trucksInArea;
  }
}
```

---

```json
{
  _id: ObjectId("userId"),
  name: "John Doe",
  email: "john.doe@example.com",
  location: {
    type: "Point",  // GeoJSON format for location
    coordinates: [-73.935242, 40.730610]  // [longitude, latitude]
  },
  notification_preference: "push"  // Or "email", "sms" based on user preference
}
```

```json
{
  _id: ObjectId("truckId"),
  truck_name: "Truck A",
  current_location: {
    type: "Point",  // GeoJSON format for location
    coordinates: [-73.935242, 40.730610]  // [longitude, latitude]
  },
  last_updated: ISODate("2024-12-10T12:00:00Z"),
  status: "active",  // Example status (active, inactive)
  assigned_area: ObjectId("areaId")  // Area the truck is assigned to
}
```

```json
// Create geospatial index for users collection
db.users.createIndex({ location: "2dsphere" });

// Create geospatial index for trucks collection
db.trucks.createIndex({ current_location: "2dsphere" });
```

```js
async function updateTruckLocation(truckId, newLocation) {
  const truck = await db.collection("trucks").updateOne(
    { _id: ObjectId(truckId) },
    {
      $set: {
        current_location: { type: "Point", coordinates: newLocation },
        last_updated: new Date(),
      },
    }
  );
  return truck;
}
```

```js
async function findNearbyUsers(truckId, radiusInMeters) {
  // Get truck location
  const truck = await db
    .collection("trucks")
    .findOne({ _id: ObjectId(truckId) });
  const truckLocation = truck.current_location.coordinates; // [longitude, latitude]

  // Find users within the specified radius from the truck's location
  const nearbyUsers = await db
    .collection("users")
    .find({
      location: {
        $nearSphere: {
          $geometry: {
            type: "Point",
            coordinates: truckLocation,
          },
          $maxDistance: radiusInMeters, // Distance in meters
        },
      },
    })
    .toArray();

  return nearbyUsers;
}
```
