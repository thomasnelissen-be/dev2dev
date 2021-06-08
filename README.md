# Dev2Dev - Develop a CAP application in less then an hour

## Pre-requisites
- Install latest version of nodejs: https://nodejs.org/en/download/

- Install the cds development kit globally (documentation of the SDK: ):

        > npm i -g @sap/cds-dk


- Install SQLite (only required for Windows) from http://sqlite.org/download.html

- Install yeoman and the SAPUI5 tempate globally:

        > npm i -g yo
        > npm i -g yo generator-easy-ui5

- Install VS Code, or your IDE of preference


## Create the initial project

- Use the cds development tools to create a new project called "dev2dev". Next, navigate to the created folder

        > cds init dev2dev
        > cd dev2dev
    
Your project folder has been initialized. It contains the following structure:

File / Folder | Purpose
---------|----------
`app/` | content for UI frontends go here
`db/` | your domain models and data go here
`srv/` | your service models and code go here
`package.json` | project metadata and configuration
`readme.md` | the getting started guide

- Install the dependencies:

        npm install

- Run the application:

        cds run

- This should give you a message similar to this one:

        No models found at undefined.
        Terminating cds serve!

    Your project does not define a data model yet. Let's create this in the next step.

## Define the data model

- Create a new file in the `db/` folder: `db/schema.cds`

        namespace com.flexso.dev2dev;

        entity Beers {
            key id      : Integer;
                brewery : String;
                name    : String;
                style   : String;
                abv     : Double;
                origin  : String;
                descr   : String;
                image   : String;
                ratings : Composition of many Ratings
                            on ratings.beer = $self
        }

        entity Ratings {
            key id          : UUID;
                beer        : Association to Beers;
                user        : String;
                collar      : Integer;
                opacity     : Integer;
                color       : Integer;
                smell       : {
                    fruit   : Integer;
                    malt    : Integer;
                    spice   : Integer
                };
                taste       : {
                    sweet   : Integer;
                    bitter  : Integer;
                    sour    : Integer;
                    body    : Integer;
                    acidity : Integer;
                };
                rating      : Integer;
        }

    This will define a data model with 2 entities and a relation between them.
    
    Run your application again:

        cds run
    This will give you the following output:

        [cds] - model loaded from 2 file(s):

            db\schema.cds
            node_modules\@sap\cds\common.cds


        [cds] - launched in: 897.765ms
        [cds] - server listening on { url: 'http://localhost:4004' }
        [ terminate with ^C ]


                No service definitions found in loaded models.
                Terminating cds serve!

We have defined a datamodel, but there is no service yet that exposes this datamodel. Let's go the next step.

## Define a service

- Create a new file in the `srv/`folder: `srv/service.cds`

        using {com.flexso.dev2dev as schema} from '../db/schema';

        service RatingService {

            entity Beers   as projection on schema.Beers;
            entity Ratings as projection on schema.Ratings;

        }

This cds defines an odata service called `RatingService`. It contains a projection to 2 entities which are defined in `../db/schema.cds` 

- Run your application again. You can use the `cds watch` command, it will detect changes to your source files and restart the application automatically.
    
        > cds watch

    This should give you the following output:

        --exec cds serve all --with-mocks --in-memory?

        [cds] - model loaded from 3 file(s):

        db\schema.cds
        srv\tasting.cds
        node_modules\@sap\cds\common.cds

        [cds] - using bindings from: { registry: '~/.cds-services.json' }
        [cds] - connect to sqlite db { database: ':memory:' }
        /> successfully deployed to sqlite in-memory db

        [cds] - serving TastingService { at: '/tasting' }

        [cds] - launched in: 1341.056ms
        [cds] - server listening on { url: 'http://localhost:4004' }
        [ terminate with ^C ]

    The application is now running on **http://localhost:4004**. 
    
    Open the url in any browser to explore the odata v4 service.
    This is a fully working oData V4 service. You can use postman to create/update/delete records.

## Add static data

Let's add some data to our database
- Create a new folder in the `db/`folder: `db/data/`
- Create a new file: `db/data/com.flexso.dev2dev.Beers.csv`. Copy the contents of [this file](./db/data/com.flexso.dev2dev.Beers.csv) to your own file.
- Create another file: `db/data/com.flexso.dev2dev.Ratings.csv`. Copy the contents of [this file](./db/data/com.flexso.dev2dev.Ratings.csv) to your own file.

The `cds watch` command should have automatically refreshed the application. Open the odata service again, you should get data from the service now.

## Adding additional features to the OData V4 service

### Calculate the average rating per beer

- Open srv/service.cds and extend the Beers entity with a calculated field (using SQL)

        entity Beers   as
        select from schema.Beers {
            *,
            (
                select avg(
                    rating
                ) as avg from schema.Ratings
                where
                    Ratings.beer.id = Beers.id
            ) as avgRating @(Core.Computed) : Double
        };

### Check for duplicate ratings per user/beer

- Create a new file called srv/service.js and add the following code:

        module.exports = (srv) => {
            srv.before("CREATE", "Ratings", checkDuplicate);
        }

        const checkDuplicate = async (req) => {
            const { Ratings } = cds.entities("com.flexso.dev2dev");
            let result = await req.run(SELECT.from(Ratings).where({beer_id: req.data.beer_id, user: req.data.user }));
            if (result.length > 0) {
                        req.reject(400, `Duplicate rating for user '${req.data.user}' for beer '${req.data.beer_id}'`);
            }
}

## Using CDS annotations to generate a Fiori Elements UI

- Create a new file called srv/annotations.cds and add the following contents:

        using RatingService from './service';

        annotate RatingService.Beers with @(UI : {LineItem : [

            {
                Value : name,
                Label : 'Name'
            },
            {
                Value : brewery,
                Label : 'Brewery'
            },
            {
                Value : abv,
                Label : 'ABV %'
            },
            {
                Value : avgRating,
                Label : 'Average Rating'
            }
        ]});

- Go to localhost:4004 and click on `Fiori Preview` next to the Beers entity



## Create a freestyle UI module



We are going to add a freestyle UI5 application that will consume this service. We will do this in the `app/` folder. To scaffold a UI5 project we will use a tool called yeoman:
        
    > cd app
    > yo easy-ui5

Answer the questions accordingly:
    
    ? Select your generator? generator-ui5-project
    ? What do you want to do? Create a new OpenUI5/SAPUI5 project                          [app]
    ? How do you want to name this project? dev2dev   
    ? Which namespace do you want to use? com.flexso
    ? On which platform would you like to host the application? Static webserver
    ? Which view type do you want to use? XML
    ? Where should your UI5 libs be served from? Content delivery network (SAPUI5)
    ? Would you like to create a new directory for the project? No

    create package.json 
    force .yo-rc.json  
    create .editorconfig
    create .eslintignore
    create .eslintrc
    create .gitignore
    create karma-ci.conf.js
    create karma.conf.js
    create readme.md
    create uimodule\ui5.yaml
    create uimodule\webapp\Component.js
    create uimodule\webapp\controller\BaseController.js
    create uimodule\webapp\css\style.css
    create uimodule\webapp\i18n\i18n_en.properties
    create uimodule\webapp\i18n\i18n.properties
    create uimodule\webapp\index.html
    create uimodule\webapp\manifest.json
    create uimodule\webapp\model\formatter.js
    create uimodule\webapp\model\models.js
    create uimodule\webapp\resources\img\favicon.ico
    create uimodule\webapp\test\integration\AllJourneys.js
    create uimodule\webapp\test\integration\arrangements\Startup.js
    create uimodule\webapp\test\integration\opaTests.qunit.html
    create uimodule\webapp\test\integration\opaTests.qunit.js
    create uimodule\webapp\test\testsuite.qunit.html
    create uimodule\webapp\test\testsuite.qunit.js
    create uimodule\webapp\test\integration\pages\Main.js
    create uimodule\webapp\test\integration\MainJourney.js
    create uimodule\webapp\view\MainView.view.xml
    create uimodule\webapp\controller\MainView.controller.js


    I'm all done. Running npm install for you to install the required dependencies. If this fails, try running the command yourself.

You have created a new UI5 application, it already contains one view: `MainView.view.xml`. Let's create two additional views:

    > yo easy-ui5

Answer the questions accordingly:

    ? Select your generator? generator-ui5-project
    ? What do you want to do? Add a new view to an existing project                        [newview]
    ? What is the name of the new view? Beers
    ? Would you like to create a corresponding controller as well? Yes
    ? Do you want to add an OPA5 page object? No
    ? Would you like to create a route in the manifest? Yes
    Updated file: /uimodule/webapp/manifest.json
    Created a new view.
    create uimodule\webapp\view\Beers.view.xml
    create uimodule\webapp\controller\Beers.controller.js
    conflict uimodule\webapp\manifest.json
    ? Overwrite uimodule\webapp\manifest.json? overwrite
        force uimodule\webapp\manifest.json


And again for the second view (Ratings.view.xml):

    > yo easy-ui5

    ? Select your generator? generator-ui5-project
    ? What do you want to do? Add a new view to an existing project                        [newview]
    ? What is the name of the new view? Ratings
    ? Would you like to create a corresponding controller as well? Yes
    ? Do you want to add an OPA5 page object? No
    ? Would you like to create a route in the manifest? Yes
    Updated file: /uimodule/webapp/manifest.json
    Created a new view.
    create uimodule\webapp\view\Ratings.view.xml
    create uimodule\webapp\controller\Ratings.controller.js
    conflict uimodule\webapp\manifest.json
    ? Overwrite uimodule\webapp\manifest.json? overwrite
        force uimodule\webapp\manifest.json

You can also add a model definition using yeoman:

    > yo easy-ui5

    ? Select your generator? generator-ui5-project
    ? What do you want to do? Add a new model to an existing project                       [newmodel]
    ? What is the name of your model, press enter if it is the default model?
    ? Which type of model do you want to add? OData v4
    ? Which binding mode do you want to use? TwoWay
    ? What is the data source url? /rating/
    Updated file: /uimodule/webapp/manifest.json
    conflict uimodule\webapp\manifest.json
    ? Overwrite uimodule\webapp\manifest.json? overwrite
        force uimodule\webapp\manifest.json


Update the routing settings in your `manifest.json` file:

    "routing": {
      "config": {
        "routerClass": "sap.f.routing.Router",
        "viewType": "XML",
        "viewPath": "com.flexso.dev2dev.view",
        "controlId": "fcl",
        "transition": "slide",
        "async": true
      },
      "routes": [
        {
          "name": "Beers",
          "pattern": "",
          "target": [
            "TargetBeers"
          ],
          "layout": "OneColumn"
        },
        {
          "name": "Ratings",
          "pattern": "Ratings/{beer}",
          "target": [
            "TargetBeers",
            "TargetRatings"
          ],
          "layout": "TwoColumnsMidExpanded"
        }
      ],
      "targets": {
        "TargetMainView": {
          "viewType": "XML",
          "viewLevel": 1,
          "viewId": "idAppControl",
          "viewName": "MainView"
        },
        "TargetBeers": {
          "viewType": "XML",
          "viewId": "Beers",
          "viewName": "Beers",
          "controlAggregation": "beginColumnPages"
        },
        "TargetRatings": {
          "viewType": "XML",
          "viewId": "Ratings",
          "viewName": "Ratings",
          "controlAggregation": "midColumnPages"
        }
      }
    }
  }

Let's add some content to the UI5 application:

- `/view/MainView.view.xml`: We will use a Flexible-Column Layout

        <mvc:View controllerName="com.flexso.dev2dev.controller.MainView" displayBlock="true" xmlns="sap.m" xmlns:mvc="sap.ui.core.mvc" xmlns:f="sap.f">
        <f:FlexibleColumnLayout id="fcl">
            <f:beginColumnPages></f:beginColumnPages>
            <f:midColumnPages></f:midColumnPages>
        </f:FlexibleColumnLayout>
        </mvc:View>
    

- `/view/Beers.view.xml`: Add a table containing all Beers

        <mvc:View controllerName="com.flexso.dev2dev.controller.Beers" displayBlock="true" xmlns="sap.m" xmlns:mvc="sap.ui.core.mvc">
        <App id="idAppControl">
            <pages>
            <Page title="Dev2Dev Rating App - Powered by CAPM">
                <content>
                <Table items="{/Beers}">
                    <headerToolbar>
                    <OverflowToolbar class="sapFDynamicPageAlignContent">
                        <Title title="Bieren" text="Beers" id="tableTitle" />
                    </OverflowToolbar>
                    </headerToolbar>
                    <columns>
                    <Column>
                        <Text text="Beer" />
                    </Column>
                    <Column>
                        <Text text="Style" />
                    </Column>
                    <Column>
                        <Text text="ABV" />
                    </Column>
                    <Column>
                        <Text text="Origin" />
                    </Column>
                    </columns>
                    <items>
                    <ColumnListItem type="Navigation" press=".onListItemPress">
                        <cells>
                        <ObjectIdentifier title="{brewery}" text="{name}" />
                        <ObjectIdentifier text="{style}" />
                        <ObjectNumber number="{abv}" numberUnit="%" />
                        <ObjectIdentifier text="{origin}" />
                        </cells>
                    </ColumnListItem>
                    </items>
                </Table>
                </content>
            </Page>
            </pages>
        </App>
        </mvc:View>
- `/controller/Beers.controller.js`

        sap.ui.define([
        "com/flexso/dev2dev/controller/BaseController"
        ], function (Controller) {
            "use strict";

            return Controller.extend("com.flexso.dev2dev.controller.Beers", {

                onListItemPress: function (oEvent) {
                    var beer = oEvent.getSource().getBindingContext().getObject();
                    var oRouter = this.getOwnerComponent().getRouter();
                    oRouter.navTo("Ratings", {
                        beer: beer.id
                    });

                }

            });

- `/view/Ratings.view.xml`: Detail screen with a table containing all ratings

        <mvc:View controllerName="com.flexso.dev2dev.controller.Ratings"
        xmlns="sap.uxap"
        xmlns:layout="sap.ui.layout"
        xmlns:m="sap.m"
        xmlns:f="sap.f"
        xmlns:form="sap.ui.layout.form"
        xmlns:mvc="sap.ui.core.mvc"
        xmlns:core="sap.ui.core">
        <ObjectPageLayout id="ObjectPageLayout" showTitleInHeaderContent="true" alwaysShowContentHeader="false" preserveHeaderStateOnScroll="false" headerContentPinnable="true" isChildPage="true" upperCaseAnchorBar="false">
            <headerTitle>
            <ObjectPageDynamicHeaderTitle id="headerTitle">
                <heading>
                <m:Title text="{brewery} - {name}" wrapping="true" class="sapUiSmallMarginEnd" />
                </heading>
                <navigationActions>
                <m:OverflowToolbarButton type="Transparent" icon="sap-icon://decline" press=".handleClose" tooltip="Close column" />
                </navigationActions>
            </ObjectPageDynamicHeaderTitle>
            </headerTitle>
            <headerContent>
            <m:FlexBox wrap="Wrap">
                <m:Avatar src="{image}" backgroundColor="Random" displaySize="XL" displayShape="Square" imageFitType="Contain" class="sapUiTinyMarginEnd" />
                <layout:HorizontalLayout class="sapUiSmallMarginBeginEnd">
                <layout:VerticalLayout class="sapUiSmallMarginBeginEnd">
                    <m:ObjectStatus title="Style" text="{style}" />
                    <m:ObjectStatus title="ABV" text="{abv}" />
                    <m:ObjectStatus title="Origin" text="{origin}" />
                </layout:VerticalLayout>
                <layout:VerticalLayout class="sapUiSmallMarginBeginEnd">
                    <m:RatingIndicator maxValue="5" value="{avgRating}" tooltip="Rating Tooltip" />
                </layout:VerticalLayout>
                </layout:HorizontalLayout>
            </m:FlexBox>
            </headerContent>
            <sections>
            <ObjectPageSection>
                <subSections>
                <ObjectPageSubSection>
                    <blocks>
                    <layout:VerticalLayout class="sapUiSmallMarginBeginEnd">
                        <m:Text text="{descr}" />
                    </layout:VerticalLayout>
                    </blocks>
                </ObjectPageSubSection>
                </subSections>
            </ObjectPageSection>
            <ObjectPageSection>
                <subSections>
                <ObjectPageSubSection>
                    <blocks>
                    <m:Table id="ratingsTable" items="{ path: 'ratings', parameters : {$$updateGroupId : 'ratings'} }" growing="true" growingThreshold="20">
                        <m:headerToolbar>
                        <m:OverflowToolbar class="sapFDynamicPageAlignContent">
                            <m:Title text="Ratings" />
                            <m:ToolbarSpacer />
                            <m:OverflowToolbarButton icon="sap-icon://add" tooltip="Add a rating" text="Add a rating" type="Transparent" press=".onAddRating" />
                        </m:OverflowToolbar>
                        </m:headerToolbar>
                        <m:columns>
                        <m:Column>
                            <m:Text text="Name" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Opacity" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Color" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Fruit" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Malt" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Spices" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Sweetness" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Biterness" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Acidity" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Body" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Carbon" />
                        </m:Column>
                        <m:Column>
                            <m:Text text="Rating" />
                        </m:Column>
                        </m:columns>
                        <m:items>
                        <m:ColumnListItem>
                            <m:cells>
                            <m:ObjectIdentifier text="{user}" />
                            <m:ObjectNumber number="{opacity}" numberUnit="/5" />
                            <m:ObjectNumber number="{color}" numberUnit="/5" />
                            <m:ObjectNumber number="{smell_fruit}" numberUnit="/5" />
                            <m:ObjectNumber number="{smell_malt}" numberUnit="/5" />
                            <m:ObjectNumber number="{smell_spice}" numberUnit="/5" />
                            <m:ObjectNumber number="{taste_sweet}" numberUnit="/5" />
                            <m:ObjectNumber number="{taste_bitter}" numberUnit="/5" />
                            <m:ObjectNumber number="{taste_sour}" numberUnit="/5" />
                            <m:ObjectNumber number="{taste_body}" numberUnit="/5" />
                            <m:ObjectNumber number="{taste_acidity}" numberUnit="/5" />
                            <m:RatingIndicator maxValue="5" value="{rating}" tooltip="Rating Tooltip" iconSize="12px" />
                            </m:cells>
                        </m:ColumnListItem>
                        </m:items>
                    </m:Table>
                    </blocks>
                </ObjectPageSubSection>
                </subSections>
            </ObjectPageSection>
            </sections>
        </ObjectPageLayout>
        </mvc:View>

- `/controller/Ratings.controller.js`:

        sap.ui.define([
            "com/flexso/dev2dev/controller/BaseController",
            "sap/ui/core/Fragment",

        ], function (Controller, Fragment) {
            "use strict";

            return Controller.extend("com.flexso.dev2dev.controller.Detail", {

                onInit: function () {

                    this.oRouter = this.getOwnerComponent().getRouter();

                    this.oRouter
                        .getRoute("Ratings")
                        .attachPatternMatched(this._onItemMatched, this);
                    this.oRouter
                        .getRoute("Beers")
                        .attachPatternMatched(this._onItemMatched, this);

                },

                _onItemMatched: function (oEvent) {
                    this._beerId = oEvent.getParameter("arguments").beer;
                    var sPath = "/Beers(" + this._beerId + ")";
                    this.getView().bindElement({ path: sPath, parameters: { $expand: 'ratings' } });
                },

                handleClose: function () {
                    this.oRouter.navTo("Beers");
                },

                onAddRating: function () {
                    var oContext = this.getModel().bindList("/Ratings", null, null, null, { $$updateGroupId: 'ratings' }).create({
                        beer_id: parseInt(this._beerId)
                    });
                    if (!this._ratingDialog) {
                        Fragment.load({
                            type: "XML",
                            name: "com.flexso.dev2dev.view.AddRating",
                            controller: this,
                        }).then(
                            function (oDialog) {
                                this._ratingDialog = oDialog;
                                this.getView().addDependent(this._ratingDialog);
                                this._ratingDialog.setBindingContext(oContext);
                                this._ratingDialog.open();
                            }.bind(this)
                        );
                    } else {
                        this._ratingDialog.setBindingContext(oContext);
                        this._ratingDialog.open();
                    }
                },

                onSaveRating: function () {
                    var oContext = this._ratingDialog.getBindingContext();
                    this.getView().getModel().submitBatch("ratings");
                    this._ratingDialog.close();
                    this.getView().getModel().refresh();
                },

                onCloseDialog: function () {
                    this._ratingDialog.getBindingContext().destroy();
                    this._ratingDialog.close();
                }

            });
        });

- `AddRating.fragment.xml`: a dialog to add a new rating

        <core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core" xmlns:l="sap.ui.layout" xmlns:f="sap.ui.layout.form">
            <Dialog title="Create Rating" id="createrating">
                <content>
                    <f:Form id="FormChange354" editable="true">
                        <f:layout>
                            <f:ResponsiveGridLayout labelSpanXL="3" labelSpanL="3" labelSpanM="3" labelSpanS="12" adjustLabelSpan="false" emptySpanXL="0" emptySpanL="0" emptySpanM="0" emptySpanS="0" columnsXL="1" columnsL="1" columnsM="1" singleContainerFullSize="false" />
                        </f:layout>
                        <f:formContainers>
                            <f:FormContainer>
                                <f:formElements>
                                    <f:FormElement label="Naam">
                                        <f:fields>
                                            <Input value="{user}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Helderheid">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{opacity}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Kleur">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{color}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Fruitig">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{smell_fruit}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Moutig">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{smell_malt}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Kruidig">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{smell_spice}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Zoet">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{taste_sweet}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Bitter">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{taste_bitter}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Zuur">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{taste_sour}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Body">
                                        <f:fields>
                                            <Slider enableTickmarks="true" min="0" max="5" value="{taste_body}" />
                                        </f:fields>
                                    </f:FormElement>
                                    <f:FormElement label="Rating">
                                        <f:fields>
                                            <RatingIndicator maxValue="5" value="{rating}" tooltip="Rating" />
                                        </f:fields>
                                    </f:FormElement>
                                </f:formElements>
                            </f:FormContainer>
                        </f:formContainers>
                    </f:Form>
                </content>
                <beginButton>
                    <Button text="Opslaan" type="Emphasized" press=".onSaveRating"></Button>
                </beginButton>
                <endButton>
                    <Button text="Annuleren" press=".onCloseDialog"></Button>
                </endButton>
            </Dialog>
        </core:FragmentDefinition>


Refresh your browser, you should now be able to open your /webapp/index.html file.

## Next Steps...

- Deploy your application to your cloudfoundry trial environment



# The Digit-Ale challenge

Finish the tutorial, upload it to your git repository of choise and send us the link: thomas.swolfs@flexso.com, thomas.nelissen@flexso.com

We will send a refreshing gift to the first 10 submitters :beer:
