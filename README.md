# Knowledge Graph Construction for Health, Lifestyle and Fitness Applications

## Listing 1
This is a sample of non-FHIR EMR reporting about the hypertension evaluation of a patient.
```json
{
  "clinical_examinations": [
    {
      "patient": {
        "patient_id": "001"
      },
      "patient_age": 29,
      "date": "2021-03-30",
      "hypertension": false
    }
  ]
}
```

## Listing 2
A sample of non-FHIR PHR from a wearable device, recording flights of stairs climbed in a day.
```json
{
  "data_source": "SH",
  "frequency": "day",
  "timestamp": "1623974400000",
  "data": [
    {
      "update_time": "1623989623494",
      "day_time": "20210618",
      "device_id": "XaIq54it7a",
      "type_id": "floorsClimbed",
      "data_uuid":"7685965849034eeeeee89",
      "values": [
        {
          "floor": "1.0",
          "start_time": "1623989575980",
          "end_time": "1623989594443",
          "time_offset": "UTC+0200"
        }
      ]
    }
  ]
}
```

## Listing 3
JSON FHIR generated from the input data in Listing.
```json
{
    "resourceType": "Bundle",
    "type": "transaction",
    "entry": [
        {...}, // patient
        {
            "resource": {
                "resourceType": "Observation",
                "id": "123",
                "identifier": [
                    {
                        "system": "https://.../identifier",
                        "value": "Observation/123"
                    }
                ],
                "status": "final",
                "code": {
                    "coding": [
                        {
                            "system": "http://loinc.org",
                            "code": "floors_climbed",
                            "display": "FloorsClimbed"
                        }
                    ]
                },
                "subject": { ... }, // patient's ref
                "effectivePeriod": {
                    "start": "2022-12-02T12:10:43.40498Z",
                    "end": "2022-12-02T12:10:44.795442Z"
                },
                "valueQuantity": {
                    "value": 1.0,
                    "unit": "floor(s)",
                    "system": "http://unitsofmeasure.org",
                    "code": "..."
                },
                "device": { ... } //device's ref
            }
        }
    ]
}
```

## Listing 4
Example of RML rules used to generate the HeLiFit triples.
```RML
<#FloorsClimbedDimension> a rr:TriplesMap;
  rml:logicalSource [
    rml:source "__RML_SRC__";
    rml:referenceFormulation ql:JSONPath;
    rml:iterator "$.entry[?
        (@.resource.resourceType == 'Observation' && 
        @.resource.code.coding[0].code == 'floors_climbed')
    ]"
  ];

  rr:subjectMap [
    rr:template "https://.../{resource.code.coding[0].code}/{resource.id}";
    rr:class ho:HLF426FloorsClimbedDimension;
  ];

  rr:predicateObjectMap [
    rr:predicate ho:P91hasUnit;
    rr:objectMap [
      rml:reference "resource.valueQuantity.unit";
    ]
  ];

  rr:predicateObjectMap [
    rr:predicate ho:P90hasValue;
    rr:objectMap [
      rml:reference "resource.valueQuantity.value";
    ]
  ]; .
```

## Listing 5
A sample of the HeLiFit turtle generated by MatKG.
```TTL
<https://__base/sdn/floors_climbed> a 
    <https://__base/HLF209IdentifierType>;
    <https://__base/P3hasNote> "http://loinc.org" .

<https://__base/type/E42/code/floors_climbed> a 
    <https://__base/E42Identifier>;
    <https://__base/P2hasType> <https://__base/sdn/floors_climbed>;
    <https://__base/P3hasNote> "floors_climbed" .

...

<https://__base/.../id/7685965849034eeeeee89> a         
    <https://__base/HLF331FloorsClimbedMeasurement>;
    <https://__base/L12happenedOnDevice>
        <https://__base/type/D8/id/XaIq54it7a>;
    <https://__base/P14carriedOutBy>
        <https://__base/id/user12%40puglia.gatekeeper.com>;
    <https://__base/P1isIdentifiedBy>
        <https://__base/type/E42/code/floors_climbed>;
    <https://__base/P40observedDimension> 
        <https://__base/type/HLF426/code/floors_climbed/id/7685965849034eeeeee89>;
    <https://__base/P4hasTimeSpan>
        <https://__base/type/E52/code/floors_climbed/id/7685965849034eeeeee89>.

<https://__base/type/HLF426/code/floors_climbed/id/7685965849034eeeeee89> a 
    <https://__base/HLF426FloorsClimbedDimension>;
    <https://__base/P90hasValue> "1";
    <https://__base/P91hasUnit> "floor(s)".
```

## Listing 6
The code for sedentary behaviour recognition.
```SPARQL
PREFIX sh: <https://opensource.samsung.com/projects/helifit/>

sh:IntensityType[?person, sh:sedentarybehavior]:- 
    sh:E21Person[?person], sh:HLF152AgeAssignment[?ageAss], 
    sh:P140assignedAttributeTo[?ageAss, ?person], 
    sh:P04hasAge[?ageAss, ?age], FILTER(?age<=64 && ?age>=18),
    sh:HLF44PhysicalActivity[?PA], sh:P14carriedOutBy[?PA, ?person], 
    sh:P117includes[?PA, ?PAAss], sh:E13AttributeAssignment[?PAAss],
    sh:P07personHasPAStepsTotal[?person, ?PAStepsTotal],
    sh:E52TimeSpan[?TS], sh:P4hasTimeSpan[?person, ?TS], 
    sh:P03hasSpanValue[?TS, ?recommendSpan], 
    FILTER(?PAStepsTotal <= 26775*?recommendSpan) .
```

## Listing 7
SPARQL query for the competency question "List all the physical activities performed by individuals in a particular time span."
```SPARQL
prefix rdf:     <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
prefix helifit: <https://opensource.samsung.com/projects/helifit/>

SELECT DISTINCT ?physicalAC WHERE {

  ?physicalAC rdf:type helifit:HLF49MuscleStrengtheningAnaerobics .
  ?phyActivity helifit:P14carriedOutBy ?ind .
  ?phyActivity helifit:P4hasTimeSpan  ?timeSpan .
  ?ind rdf:type helifit:E21Person .
  ?ind helifit:P1isIdentifiedBy ?userID .
  ?timeSpan helifit:EP9effectiveDatatime ?time .
  FILTER (?time > "2022-01-28T17:17:48Z"^^xsd:dateTime && 
  ?time <= "2022-02-28T17:17:48Z"^^xsd:dateTime)
}
```

