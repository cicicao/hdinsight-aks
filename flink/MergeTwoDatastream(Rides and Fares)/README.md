The goal of this exercise is to join together the TaxiRide and TaxiFare records for each ride.

For each distinct rideId, there are exactly three events:

a TaxiRide START event
a TaxiRide END event
a TaxiFare event (whose timestamp happens to match the start time)
The result should be a DataStream<RideAndFare>, with one record for each distinct rideId. Each tuple should pair the TaxiRide START event for some rideId with its matching TaxiFare.
