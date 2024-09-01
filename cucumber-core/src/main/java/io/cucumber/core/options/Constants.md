* `cucumber.ansi-colors.disabled`
  * disable ansi colors | output
  * NOT supported | ALL terminals
* `cucumber.properties`
  * fileName of cucumber properties
* `cucumber.filter.name`
  * == reggex pattern / -- filter -- scenarios to be executed
  * by default, ALL scenarios are executed
* `cucumber.features=pathToFeature1, pathToFeature2, ...`
  * `pathToFeatureX`
    * `PATH[.feature[:LINE]*]` or
    * `URI[.feature[:LINE]*]` or
    * `@PATH`
    * _Examples:_
      * _Example1:_ "src/test/resources/features" == "src/test/resources/features"
      * _Example2:_ "classpath:com/example/application" == "com.example.application"
      * _Example3:_ TODO:
* TODO:
* `cucumber.glue=gluePath1, gluePath2, ..`
  * `GluePath`
* TODO: