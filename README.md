# Content IG Walkthrough

This repository provides a walkthrough of building a FHIR content implementation guide (content IG). A content IG is a FHIR implementation guide that primarily contains knowledge artifacts such as decision support rules and quality measures. By using the FHIR publication toolchain, these artifacts are made available as FHIR resources, published within websites for documentation and dissemination, and distributed as part of NPM packages for implementation.

This walkthrough consists of 5 main steps:

1. [**Setup**](#basic-setup): Setting up the IG and getting a basic build
2. [**Library**](#adding-a-library): Including a simple Library for a specific recommendation
3. [**PlanDefinition**](#adding-a-plandefinition): Adding a PlanDefinition to surface the recommendation as a decision support rule
4. [**Test Cases**](#adding-test-cases): Adding test cases to test the rule
5. [**Validation**](#validation-with-cqf-ruler-and-cds-hooks): Validating the content works as expected via the CDS Hooks Sandbox


## Basic Setup

The first step is to get a local [clone](https://docs.github.com/en/free-pro-team@latest/github/creating-cloning-and-archiving-repositories/cloning-a-repository) of the walkthrough repository.

To open a terminal in VS Code:

***Prerequisites***

- install a `git` client

***Steps***

- In the top menu, click `Terminal`
- click `New Terminal`
- in the `Terminal`, run

```bash
git clone https://github.com/cqframework/content-ig-walkthrough.git
```

Once you have a local clone, you'll need to build:

### Dependencies

Before you're able to build this IG you'll need to install several dependencies

#### Java / IG Publisher

Go to [https://adoptopenjdk.net/](https://adoptopenjdk.net/) and download the latest (version 8 or higher) JDK for your platform, and install it.

There are scripts in this repository that will download and run the latest HL7 IG Publisher.

Please make sure that the Java bin directory is added to your path.

This project includes scripts that will automatically download the correct version of the IG-Publisher.

Documentation for the IG-Publisher is available at [https://confluence.hl7.org/display/FHIR/IG+Publisher+Documentation](https://confluence.hl7.org/display/FHIR/IG+Publisher+Documentation).

#### Ruby / Jekyll

Jekyll requires Ruby version 2.1 or greater. Depending on your operating system, you may already have Ruby bundled with it. Otherwise, or if you need a newer version, go to [https://www.ruby-lang.org/en/downloads/](https://www.ruby-lang.org/en/downloads/) for directions.

Jekyll

Go to [https://jekyllrb.com](https://jekyllrb.com) and follow the instructions there, for example gem install jekyll bundler. The end result of this should be that the binary "jekyll" is now in your path.

### Build

Once you have the dependencies installed, you can build the IG by first updating the publisher:

```
_updatePublisher
```

Once the publisher has been downloaded to your local environment, you can build the IG:

```
_genonce
```

Whenever you make changes to the source content, just rerun the `_genonce` script to rebuild the IG. The IG output will be in the output folder. Using a browser, open the `index.html` page to see the IG.

NOTE: This walkthrough uses "contentig" as the name of the implementation guide. The source ImplementationGuide resource is in the [contentig.xml](input/contentig.xml) file, and the [ig.ini](ig.ini) file refers to it. To change the name of the ig, be sure to update all references to `contentig` within the ImplementationGuide resource. Note that this name serves as part of the basis for the canonical URL for your IG, which is used to uniquely identify the implementation guide and all the resources published within it, so it's important to choose your canonical URL appropriately. For more information see the [Canonical URLs](https://www.hl7.org/fhir/references.html#canonical) discussion topic in the FHIR specification.

## Adding a Library

The next step in this walkthrough is to build the recommendation logic as expressions of [Clinical Quality Language](http://cql.hl7.org) (CQL). CQL is a high-level, author-friendly language that is used to express the logic used in clinical reasoning artifacts such as decision support rules, quality measurement population criteria, and public health reporting criteria.

### VSCode CQL Support

To validate and test CQL, use the [VSCode CQL Plugin](https://github.com/cqframework/vscode-cql). Follow the instructions there to install the plugin, then open VS Code on the root folder of this content IG walkthrough.

### Recommendation A2 - Iron and Folic Acid Supplements

For this walkthrough, we'll be putting together a recommendation from the World Health
Organization (WHO) [recommendations on antenatal care for a positive pregnancy experience](https://www.who.int/reproductivehealth/publications/maternal_perinatal_health/anc-positive-pregnancy-experience/en/).

Specifically, we'll be looking at recommendation A2.1, Iron and Folic Acid Supplements:

> RECOMMENDATION A.2.1: Daily oral iron and folic acid supplementation with 30 mg to
> 60 mg of elemental iron and 400 &mu;g (0.4 mg) folic acid is recommended for pregnant
> women to prevent maternal anaemia, puerperal sepsis, low birth weight, and preterm birth.
> (Recommended)

To create a production-ready implementation of this recommendation is a significant undertaking, but for
the purposes of this walkthrough, we'll make some explicit assumptions to simplify the process so we
can focus on the techniques for expression, distribution, and evaluation of decision support rules
using a FHIR content IG. Specifically, we'll assume:

1. Use of the [International Patient Summary](http://hl7.org/fhir/uv/ips/) to characterize pregnancy status
2. Decision support is taking place in the context of a regularly scheduled contact with the pregnant mother
3. We'll focus only on the standard cases for anaemia, and preventing anaemia
4. We'll focus exclusively on presenting the recommendation, keeping followup and compliance out of scope for this walkthrough
5. We'll use terminology from the International Patient Summary, or from OpenMRS
6. We'll assume delivery of this decision support via a CDS Hooks integration

#### Pseudo-code for the Logic

First, we break down the recommendation into "pseudo-code" to get to a functional description of what
needs to happen in order to apply the recommendation as part of a contact with the mother:

> On every contact,
>   if anaemia detected
>     recommend 120 mg of elemental iron and 400 &mu;g of folic acid daily (Recommendation A.2.1)
>   else
>     recommend 30 to 60 mg of elemental iron and 400 &mu;g of folic acid, daily (Recommendation A.2.1)

Basically, if the mother has anaemia, recommend 120 mg of elemental iron and 0.4 mg of folic acid daily, otherwise, recommend the standard 30 to 60 mg of elemental iron and 0.4 mg of folic acid daily.

#### Data Elements

The next step is to characterize the data elements involved in the expression of the logic. The most obvious one is the definition for anaemia. Following the WHO guideline, we use the following method to determine whether the mother has anaemia:

> Has Anaemia
>   Hb Concentration < 11 g/dL and Gestational Age < 12 weeks or Gestational Age > 28 weeks
>   Hb Concentration < 10.5 g/dL and Gestational Age between 13 weeks and 27 weeks

To express this calculation, we then need the following elements:

* Hemoglobin (Hb) concentration
* Gestational age in weeks
* Estimated due date (to calculate gestational age)
* Pregnancy status

#### Pregnancy Status

For pregnancy status, we'll make use of the [Pregnancy Status](http://hl7.org/fhir/uv/ips/StructureDefinition-Observation-pregnancy-status-uv-ips.html) profile in the International Patient Summary. This profile requires the use of a specific LOINC code for pregnancy status, for both the observation code and value:

```
code "Pregnancy status": '82810-3' from LOINC display 'Pregnancy status'
code "Pregnancy status - Pregnant": 'LA15173-0' from LOINC display 'Pregnant'
```

With these terminology elements defined, we can reference the most recent pregnancy observation with a simple query:

```
define Pregnant:
    "Pregnancy Status" ~ "Pregnancy status - Pregnant"

define "Pregnancy Status":
  FHIRHelpers.ToConcept(
    First(
      [Observation: "Pregnancy status"] O
        where O.status = 'final'
          and O.issued 1 year or less before Today()
        sort by effective.value
    ).value
  )
```

#### Estimated Due Date

The next element we need is the estimated due date, and again, we'll make use of the International Patient Summary: [Estimated Due Date](http://hl7.org/fhir/uv/ips/StructureDefinition-Observation-pregnancy-edd-uv-ips.html). This profile identifies a possible set of delivery date estimation methods, which we reference using a `valueset` declaration in the CQL:

```
valueset "Pregnancy Expected Delivery Date Method - IPS": 'http://hl7.org/fhir/uv/ips/ValueSet/edd-method-uv-ips'
```

With this value set, we can now define a similar query for the most recent estimated due date:

```
define "Estimated Due Date":
  FHIRHelpers.ToDateTime(
    First(
      [Observation: "Pregnancy Expected Delivery Date Method - IPS"] O
        where O.status = 'final'
          and O.issued 1 year or less before Today()
        sort by FHIRHelpers.ToDateTime(effective as FHIR.dateTime) desc
    ).value
  )
```

#### Gestational Age in Weeks

Next, we'll use the estimated due date to calculate a gestational age in weeks:

```
define "Gestational Age in Weeks":
  weeks between ("Estimated Due Date" - 280 days) and Today()
```

#### Hemoglobin (Hb) Concentration

The next data element we need is the Haemoglobin concentration, which, according to WHO guidelines, can be determined from a number of tests, as defined by the `Haemoglobin Tests` value set:

```
valueset "Haemoglobin Tests": 'http://fhir.org/guides/who/anc-cds/ValueSet/haemoglobin-tests'
```

```
define "Hb Concentration":
  FHIRHelpers.ToQuantity(
    First(
      ["Observation": Common."Haemoglobin Tests"] O
  		  where O.status = 'final'
          and O.issued 1 year or less before Today()
        sort by FHIRHelpers.ToDateTime(effective as FHIR.dateTime) descending
    ).value
  )
```

#### Has Anemia

Now that we have the required data elements, we can express the `Has Anaemia` data element with a simple calculation, based on the WHO guidelines:

```
define "Has Anaemia":
  if "Gestational Age in Weeks" between 13 and 27 then
    "Hb Concentration" < 10.5 'g/dL'
  else
    "Hb Concentration" < 11 'g/dL'
```

### The Library resource

Now that we have the [CQL source defined](input/cql/ANCRecommendationA2.cql), we can package it for inclusion in the IG using a [Library](http://hl7.org/fhir/library.html) resource. The Library resource can be used for a variety of knowledge artifact packaging purposes, but in this specific case, we'll use it as a content library to store the source and compiled [ELM](https://cql.hl7.org/04-logicalspecification.html) for the CQL.

First, we set up the [Library](input/resources/Library-ANCRecommendationA2.json), specifying basic metadata information like the `name`, `title`, and `status`:

```
{
  "resourceType": "Library",
  "id": "ANCRecommendationA2",
  "name": "ANCRecommendationA2",
  "title": "WHO Antenatal Care Guidelines Recommendation A2 Logic",
  "status": "active",
  "experimental": true,
  "type": {
    "coding": [
      {
        "system": "http://hl7.org/fhir/codesystem-library-type.html",
        "code": "logic-library"
      }
    ]
  },
  "content": [
    {
      "id": "ig-loader-ANCRecommendationA2.cql"
    }
  ]
}
```

In addition, we specify that it's a `logic-library`, meaning it specifically carries logic for evaluation as part of a knowledge artifact, and we then tell the publisher how to find the source CQL using the `content.id` element, setting it to the text `ig-loader-` followed by the name of the CQL source file. The publisher will then translate the CQL source and populate the rest of the metadata in the library (such as parameters, data requirements, and dependencies), and include the published library as a resource in the implementation guide.

## Adding a PlanDefinition

The next step is to establish the decision support rule. We do this using a [PlanDefinition](http://hl7.org/fhir/plandefinition.html) resource in FHIR. PlanDefinition is a flexible resource, but the approach we're using here is to use it to represent an Event-Condition-Action rule (ECA rule), which says:

* When some _event_ occurs
* If some _condition_ is true
* Perform some _action_

### Event

The event in our case is an encounter with the patient, the condition is that the patient is pregnant and has anaemia, and the action is to recommend iron and folic acid supplements.

The [plandefinition-ANCRecommendationA2](input/resources/plandefinition-ANCRecommendationA2.xml) resource sets this up. First, the _event_, specified in the `action.trigger` element of the PlanDefinition:

```
<trigger>
  <type value="named-event"/>
  <name value="patient-view"/>
</trigger>
```

Specifically, we're using the `patient-view` hook defined in CDS Hooks as a simple way to say that we want the logic to be invoked when the patient's chart is opened.

### Condition

Next, we need to express the _condition_, specified using the `action.condition` element of the PlanDefinition:

```
<condition>
  <kind value="applicability"/>
  <expression>
    <language value="text/cql-identifier"/>
    <expression value="Is Recommendation Applicable"/>
  </expression>
</condition>
```

In the CQL, we define the `Is Recommendation Applicable` expression as the inclusion criteria for this recommendation; in this case, just that the mother is pregnant:

```
define "Is Recommendation Applicable":
  "Pregnant"
```

The PlanDefinition has a reference to the ANCRecommendationA2 library, so that any expression-valued element can refer to the name of an expression deined within that library:

```
<library value="http://somewhere.org/fhir/uv/contentig/Library/ANCRecommendationA2"/>
```

### Action

And finally, the _action_ we want to take, in this case to display a CDS Hooks _card_ to the user. For the [card attributes](https://cds-hooks.hl7.org/1.0/#card-attributes), we want to specify:

* summary
* detail
* indicator
* source

First, we set up the `source` using the `action.documentation` element to provide a link back to the WHO guideline:

```
<documentation>
    <type value="documentation"/>
    <display value="WHO recommendations on antenatal care for a positive pregnancy experience"/>
    <url value="https://www.who.int/reproductivehealth/publications/maternal_perinatal_health/anc-positive-pregnancy-experience/en/"/>
</documentation>
```

Next, in the CQL, we can build the dynamic values directly:

```
define "Get Card Summary":
  if "Has Anaemia" then
    'Recommend 120 mg elemental iron and 0.4 mg folic acid, daily'
  else
    'Recommend 30-60 mg elemental iron and 0.4 mg folic acid, daily'

define "Get Card Detail":
  if "Has Anaemia" then
  'Daily elemental iron should be increased to 120 mg, and daily dose of 400 ug (0.4 mg) until her Hb concentration rises to normal'
  else
    'Daily elemental iron of between 30m and 60mg, and daily dose of 400 ug (0.4 mg) of folic acid is recommended for pregnant women'

define "Get Card Indicator":
  if "Has Anaemia" then
    'warning'
  else
    'info'
```

And then use the `dynamicValue` element of the PlanDefinition to reference these expressions:

```
<dynamicValue>
  <path value="action.description"/>
  <expression>
    <language value="text/cql-identifier"/>
    <expression value="Get Card Detail"/>
  </expression>
</dynamicValue>
<dynamicValue>
  <path value="action.title"/>
  <expression>
    <language value="text/cql-identifier"/>
    <expression value="Get Card Summary"/>
  </expression>
</dynamicValue>
<dynamicValue>
  <path value="action.extension"/>
  <expression>
    <language value="text/cql-identifier"/>
    <expression value="Get Card Indicator"/>
  </expression>
</dynamicValue>
```

The event-condition-action processing works by evaluating the condition expression, and then constructing a CDS Hooks card if the condition evaluates to true.

## Adding Test Cases

To test that the logic behaves as expected, we can define tests along-side the content. The atom plugin expects a [`tests`](input/tests) folder to be defined within the IG `input` directory. Within this folder, the plugin expects a folder named the same as the CQL library under test, [`ANCRecommendationA2`](input/tests/ANCRecommendationA2) in this case, and within that folder, any number of test cases, defined as patient test cases with all their associated data.

For this walkthrough, we've defined two test cases, a `mom-with-anaemia` and a `mom-without-anaemia`, and provided the Patient and Observation resources required.

To execute the tests, open the ANCRecommendationA2.cql file, right-click anywhere in the editor window and select CQL->Execute from the right-click menu. A new editor window will appear with the results of running the tests, displaying the result of evaluating each top-level item defined in the library.

These test cases can be bundled using tooling defined in the [CQF Tooling](https://github.com/cqframework/cqf-tooling) repository.

## Validation with CQF Ruler and CDS Hooks

As a final step in the walkthrough, we'll validate that this content is functional by loading it into a [CQF Ruler](https://github.com/dbcg/cqf-ruler) and then testing it with the [CDS Hooks Sandbox](https://sandbox.cds-hooks.org).

### Loading the content

Once the IG has been built successfully, the content can be loaded into a CQF Ruler.

We need to load:

1. The [ANCRecommendationA2 Library](http://build.fhir.org/ig/cqframework/content-ig-walkthrough/Library-ANCRecommendationA2.json)
2. The [ANCRecommendationA2 PlanDefinition](http://build.fhir.org/ig/cqframework/content-ig-walkthrough/PlanDefinition-ANCRecommendationA2.json)
3. The [Estimated Due Date Method ValueSet](input/vocabulary/ValueSet/external/valueset-edd-method-uv-ips.json)
4. The [Haemoglobin Tests ValueSet](input/vocabulary/ValueSet/external/valueset-haemoglobin-tests.json)

NOTE: The IPS and WHO ANC value sets are defined in those implementation guides, we have copied them here for ease of reference, and placed them in an `external` folder in the vocabulary. This structure allows the VS Code extension to reference the vocabulary without explicitly publishing them as part of this implementation guide. A better distribution mechanism (currently underway) is to use the NPM package, rather than requiring the value sets to be copied.

We can use a REST client VSCode extension to do this.

Prerequisites:

- install the ThunderClient VS Code extension

In the ThunderClient extension:

- Open the `IG Walkthrough` Collection
- Run all the requests in Setup/Content

Loading these into a running CQF Ruler, you can use the [CDS Hooks Discovery endpoint](https://cds-hooks.hl7.org/1.0/#discovery) to see the ANCRecommendationA2 service:

https://cloud.alphora.com/sandbox/r4/cds/cds-services

This returns the service description:

```
{
  "services": [
    {
      "hook": "patient-view",
      "name": "PlanDefinition_ANCRecommendationA2",
      "title": "PlanDefinition - WHO ANC Guideline Recommendation A.2",
      "id": "ANCRecommendationA2",
      "prefetch": {
        "item1": "Patient?_id\u003d{{context.patientId}}",
        "item2": "Observation?subject\u003dPatient/{{context.patientId}}\u0026code\u003dhttp://loinc.org|11778-8,http://loinc.org|11779-6,http://loinc.org|11780-4",
        "item3": "Observation?subject\u003dPatient/{{context.patientId}}\u0026code\u003dhttp://openmrs.org/concepts|1019AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA,http://openmrs.org/concepts|165395AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA,http://openmrs.org/concepts|165396AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
        "item4": "Observation?subject\u003dPatient/{{context.patientId}}\u0026code\u003dhttp://loinc.org|82810-3"
      }
    }
  ]
}

#WARNING
If your service is not showing and you are using the CQF Ruler, there is a known bug where CDS PlanDefinition resources added after the discovery endpoint has been called (ever) will not get loaded.  You must post all CDS PlanDefinition resources before discovery is called.  This means if you're loading a CDS PlanDefinition to an instance where discovery has already been called, you must kill the service and start a new one (and reload all the CDS PlanDefinition resources again).
 
```
Note the [_prefetch templates_](https://cds-hooks.hl7.org/1.0/#prefetch-template) defined in the service; these are inferred directly from the CQL library, based on the [_retrieve_](https://cql.hl7.org/02-authorsguide.html#retrieve) statements.

### Loading the Patient Data

Next we need to load the Patient data into the sandbox:

* [Patient](input/tests/mom-with-anaemia/Patient/patient-mom-with-anaemia.json)
* [Pregnancy Status Observation](input/tests/mom-with-anaemia/Observation/observation-mom-with-anaemia-pregnancy-status.json)
* [Estimated Due Date Observation](input/tests/mom-with-anaemia/Observation/observation-mom-with-anaemia-edd.json)
* [Haemoglobin Observation](input/tests/mom-with-anaemia/Observation/observation-mom-with-anaemia-hb.json)

We can use a REST client VSCode extension to do this.

Prerequisites:

- install the ThunderClient VS Code extension

In the ThunderClient extension:

- Open the `IG Walkthrough` Collection
- Run all the requests in Setup/Test Data

### Configuring the CDS Hooks Sandbox

Next, we configure the CDS Hooks sandbox to use the cds-sandbox as the FHIR server, and as the CDS Hooks service. To configure the cds-sandbox as the FHIR Server, navigate to the CDS Hooks sandbox:

http://sandbox.cds-hooks.org/

In the upper right-hand corner of the page, click the settings gear to bring down the settings menu and select `Change FHIR Server`. In the dialog that is displayed, Enter the URL for the CDS Sandbox FHIR server:

https://cloud.alphora.com/sandbox/r4/cds/fhir

and then click Next. In the patient dialog that comes up, enter the Patient ID:

`mom-with-anaemia`

and then click Next.

To configure the sandbox to call the ANCRecommendationA2 service, click the settings gear again and select `Add CDS Service`. In the dialog that is displayed, enter the URL for the CDS Sandbox Discovery endpoint:

https://cloud.alphora.com/sandbox/r4/cds/cds-services

and then click Next. The CDS Hooks sandbox will then call that service with the mom-with-anaemia patient, and return the recommendation. To see the request/response, click the Select a Service drop down in the CDS Developer panel and select ANCRecommendationA2. This will display the actual CDS Hooks request that was sent, and the CDS Hooks response that came back.
