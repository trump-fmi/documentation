
# Dependencies
To compile the label pipeline the following libraries have to be
installed on the system.
```
cgal
cppunit
boost
```
The compile process can be started with the `install.sh` script.

# Usage
To use the pipeline you need a pbf file containing the openstreetmap
data as well as a config file which is explained later in this
document. The pipeline can then be executed by:
``` shell
./pipeline.sh ../config/my.conf stuttgart-regbez-latest.osm.pb
```
The output is a ".ce" file in the same directory which is named as
determined in the config file.
 
# Config 
The config is written in the json format. In the following there will
always be presentetd part of the config and a description afterwards.

``` json
{
	"labeling_name" : "some labeling",
		"description" : "Text describing what the labeling contains",
```
The config starts with the name of the output file and a description.

``` json
		"label_split" : {
			"split_bound" : 15,
			"split_chars" : [
				" ", "-", "/"
			]
		},
```

The label_split attribute defines how long a label has to be to be
splitted and which characters can be used as a splitting bound.

``` json
		"font" : {
			"name" : "Dejavu Sans",
			"ttf-path" : "/usr/share/fonts/TTF/DejaVuSans.ttf"
		},
```

The used font is important because the pipeline uses it to calculate
the width of the labels. This should obviously be the same font used
on the frontend.

``` json
		"filter" : {
			"type" : "or",
			"operands" : [
			{
				"type" : "and",
				"operands" : [
					"place", "name"
				]
			},
			"amenity"
			]
		},
```
 
 The filter defines which OSM IDs should be used. It filters on the
 existence of tags. As the contents of the name tag is used as the
 label, one should probably always for it. By the logical conjunctions
 "and" and "or" the filters can be build up to complex queries.
 
``` json

		"mapping" : {
			"level": "OSM_POI",
			"sublevels": [
			{
				"level": "SETTLEMENT",
				"constraints": [
				{
					"tag": "place"
				}
				]
			}
			]
		}
```

In the mapping a definitions is made how each label should be
displayed by sorting them into levels . All labels belong to the
toplevel (here "OSM_LEVEL"). In each sublevel new constraints can be
added and only labels conforming to these constraints become part of
the sublevel. The "tag" constraint means that the labels have to
contain the specified tag.

``` json
			"constraints": [
			{
				"tag": "place",
				"equals": "city"
			}
			],
```

With the equals constraint the value of a tag can be specified.

``` json

			"constraints": [
			{
				"tag":"population",
				"greater": 5000000
			}
			],
```

Numerical values can be filter by size with the greater constraint.

``` json
			{
				"level": "LARGE_TOWN",
				"constraints": [
				{
					"tag": "population",
					"greater": 100000
				}
				],
				"factor":20
			},
```


The labels of a certian level can be increased in size by using the
factor attribute.


``` json

				"level": "SCHOOL",
				"constraints": [
				{
					"tag": "amenity",
					"equals": "education"
				},
				{
					"tag": "amenity",
					"equals": "language_school"
				},
				{
					"tag": "amenity",
					"equals": "school"
				}
				],
				"factor": 8,
				"icon": "school"
```

 A level can of course contian more than one constraint. A label
 belongs to the level if one of the given constraints match. Instead
 of a label an icon can be specified. The text can be chosen
 freely. It is the responsibility of the frontend to interpret the string.
