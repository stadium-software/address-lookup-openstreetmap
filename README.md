# OpenStreetMap Address Lookup

A search API looks up a location from a textual description or address. 

This module integrates the free Nominatim API of OpenStreetMap. Please make sure you adhere to the [usage policies](https://operations.osmfoundation.org/policies/nominatim/). 
https://github.com/user-attachments/assets/8284de7b-7015-49fd-af5a-355c7c66f75a

## Version 
1.0.1 Updated for Stadium 6.12

1.1 Integrated CSS into script; added icon color to css variables

# Setup

## Requirements
The module calls the https://nominatim.openstreetmap.org/search API to enable address lookups and requires an Internet connection

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Global Script
1. Create a Global Script called "OpenStreetMapLookup"
2. Add the input parameters below to the Global Script
   1. ClassName
   2. CountryCodes
   3. MaxResultsCount
3. Drag a *JavaScript* action into the script
4. Add the Javascript below unchanged into the JavaScript code property
```javascript
/* Stadium Script v1.1 https://github.com/stadium-software/address-lookup-openstreetmap */
let classname = ~.Parameters.Input.ClassName;
let limit = ~.Parameters.Input.MaxResultsCount;
let countryCodes = ~.Parameters.Input.CountryCodes;
if (Array.isArray(countryCodes) && countryCodes.length > 0) {
    countryCodes = "&countrycodes=" + countryCodes.join(",");
} else {
    countryCodes = "";
}
if (isNaN(limit)) limit = 10;
if (!classname) {
     console.error("The 'ClassName' parameter is required");
     return false;
}
let lookupContainer = document.querySelectorAll("." + classname);
if (lookupContainer.length == 0) {
    console.error("The class '" + classname + "' is not assigned to any container");
    return false;
} else if (lookupContainer.length > 1) {
    console.error("The class '" + classname + "' is assigned to multiple containers");
    return false;
} else { 
    lookupContainer = lookupContainer[0];
}
lookupContainer.classList.add("address-lookup-osm");
let lookupInput = lookupContainer.querySelector("input[type='text']");
let scope = this;
let getObjectName = (obj) => {
    let objname = obj.id.replace("-container","");
    do {
        let arrNameParts = objname.split(/_(.*)/s);
        objname = arrNameParts[1];
    } while ((objname.match(/_/g) || []).length > 0 && !scope[`${objname}Classes`]);
    return objname;
};
let delay = (function () {
    let timer = 0;
    return function (callback, ms) {
        lookupContainer.classList.add("looking-up");
        clearTimeout(timer);
        timer = setTimeout(callback, ms);
    };
})();
lookupInput.addEventListener("keyup", function () {
    delay(lookup, 1000);
});
lookupInput.addEventListener("paste", function () {
    lookup();
});
loadCSS();
removeResults();

function lookup() {
    let val = getDMValues(lookupContainer, "Text");
    if (val && val != '') {
        removeResults();
        let address = encodeURI(val);
        fetch("https://nominatim.openstreetmap.org/search?addressdetails=1&q=" + address + "&format=jsonv2&limit=" + limit + countryCodes, {
            method: "get",
        })
            .then((response) => response.json())
            .then(function (jsonData) {
                lookupContainer.classList.remove("looking-up");
                let resultsContainer = document.createElement("div");
                resultsContainer.classList.add("results-container");
                lookupContainer.appendChild(resultsContainer);
                for (let i = 0; i < jsonData.length; i++) {
                    let result = document.createElement("div");
                    result.classList.add("lookup-result");
                    result.setAttribute("address", JSON.stringify(jsonData[i].address));
                    result.textContent = jsonData[i].display_name;
                    result.addEventListener("click", populateInput);
                    resultsContainer.appendChild(result);
                }
            })
            .catch((err) => {
                lookupContainer.classList.remove("looking-up");
                console.log(err);
            });
    } else { 
        lookupContainer.classList.remove("looking-up");
    }
}
document.addEventListener("click", (e) => {
    if (!e.target.classList.contains("results-container") && document.querySelector(".results-container")) {
        removeResults();
    }
});
function removeResults() {
    let allResults = document.querySelectorAll(".results-container");
    if (allResults.length > 0) {
        for (let i = 0; i < allResults.length; i++) {
            allResults[i].remove();
        }
    }
}
function populateInput(e) {
    let res = e.target;
    setDMValues(lookupContainer, "Text", res.textContent);
    scope.AddressSelectEventHandler(res.getAttribute("address"));
    removeResults();
}
function getDMValues(ob, property) {
    let obname = getObjectName(ob);
    return scope[`${obname}${property}`];
}
function setDMValues(ob, property, value) {
    let obname = getObjectName(ob);
    scope[`${obname}${property}`] = value;
}
function loadCSS() {
    let moduleID = "stadium-address-lookup";
    if (!document.getElementById(moduleID)) {
        let cssMain = document.createElement("style");
        cssMain.id = moduleID;
        cssMain.type = "text/css";
        cssMain.textContent = `
.address-lookup-osm {
    position: relative;
    .results-container {
        position: absolute;
        overflow: auto;
        width: max-content;
        top: 100%;
        background-color: var(--address-lookup-results-container-background-color, var(--BODY-BACKGROUND-COLOR));
        border: 1px solid var(--address-lookup-results-container-border-color, var(--BODY-FONT-COLOR));
    }
    .lookup-result {
        cursor: pointer;
        padding: 0.6rem;
        font-size: var(--address-lookup-results-item-font-size, var(--DATA-GRID-HEADER-CELL-FONT-SIZE));
        color: var(--address-lookup-results-item-font-color, var(--BODY-FONT-COLOR));
    }
    .lookup-result:hover {
        background-color: var(--address-lookup-results-item-hover-color, var(--LIGHT-GREY));
    }
}
.address-lookup-osm.looking-up:after {
    content: "";
    background-color: var(--address-lookup-textbox-active-icon-color, var(--BODY-FONT-COLOR));
    mask-image: var(--address-lookup-textbox-active-icon, url("data: image/svg+xml, %3Csvg xmlns='http://www.w3.org/2000/svg' width='1em' height='1em' viewBox='0 0 24 24' %3E%3Ccircle cx='4' cy='12' r='3' fill='currentColor'%3E%3Canimate id='SVG7x14Dcom' fill='freeze' attributeName='opacity' begin='0;SVGqSjG0dUp.end-0.25s' dur='0.75s' values='1;.2'/%3E%3C/circle%3E%3Ccircle cx='12' cy='12' r='3' fill='currentColor' opacity='.4'%3E%3Canimate fill='freeze' attributeName='opacity' begin='SVG7x14Dcom.begin+0.15s' dur='0.75s' values='1;.2'/%3E%3C/circle%3E%3Ccircle cx='20' cy='12' r='3' fill='currentColor' opacity='.3'%3E%3Canimate id='SVGqSjG0dUp' fill='freeze' attributeName='opacity' begin='SVG7x14Dcom.begin+0.3s' dur='0.75s' values='1;.2'/%3E%3C/circle%3E%3C/svg%3E"));
    mask-repeat: no-repeat;
    mask-position: center;
    mask-size: contain;
    position: absolute;
    width: calc(var(--address-lookup-textbox-active-icon-size, 2.6rem) + .4rem);
    height: 100%;
    right: calc((var(--address-lookup-textbox-active-icon-size, 2.6rem) / 2) + 0.8rem);
    top: 0;
}
.address-lookup-osm.looking-up input {
    padding-right: calc(var(--address-lookup-textbox-active-icon-size, 2.6rem) + 1.2rem);
}
html {
    min-height: 100%;
    font-size: 62.5%;
}
        `;
        document.head.appendChild(cssMain);
    }
}
```

## Custom Event Handler Script
1. Add a script to the page and name it "AddressSelectEventHandler"
2. Add an input parameter to the script and call it "Data"
3. Drag a *Notification* into the script (so you can see what the object you will process looks like)
4. Assign the input parameter called "Data" to the *Message* property of the *Notification*
5. When the user selects an address, this script will be called and you can process the selected address as required

![](images/CustomEventHandler.png)

**Example Response Object**
```json
{   
    "road":"Mountain View Road",
    "suburb":"Maitland Garden Village",
    "city":"Cape Town",
    "county":"City of Cape Town",
    "state":"Western Cape",
    "ISO3166-2-lvl4":"ZA-WC",
    "postcode":"7450",
    "country":"South Africa",
    "country_code":"za"
}
```

## Page
1. Drag a *TextBox* control to the page
2. Add a unique class to the control *Classes* property

## Page.Load
1. Drag the "OpenStreetMapLookup" script to the Page.Load event handler
2. Complete the input parameters
   1. ClassName: The unique class you added to the *TextBox* classes property above
   2. MaxResultsCount (int): By default the search results list is limited to 10 items. Add another number if you wish to increase or decrease this limit
   3. CountryCodes: By default the search will be performed across all countries in the world. If you wish to limit the countries from which search results are retrieved, create a *List* of countries to search using the [ISO 3166-1 country codes](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) and provide the *List* in this parameter. 

**CountryCodes List Example Values**
```json
["za","gb"]
```

## CSS
Variables exposed in the [*address-lookup-variables.css*](address-lookup-variables.css) file can be [customised](#customising-css).

### Customising CSS
1. Open the CSS file called [*address-lookup-variables.css*](address-lookup-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit
3. Add the [*address-lookup-variables.css*](address-lookup-variables.css) to the "CSS" folder in the EmbeddedFiles (overwrite)
4. Paste the link tag below into the *head* property of your application
```html
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/address-lookup-variables.css">
``` 
5. Stadium 6.12+ users can comment out any variable they do not wish to customise

**NOTE: Do not change any of the CSS in the 'address-lookup.css' file**

## Upgrading Stadium Repos
Stadium Repos are not static. They change as additional features are added and bugs are fixed. Using the right method to work with Stadium Repos allows for upgrading them in a controlled manner. 

How to use and update application repos is described here: [Working with Stadium Repos](https://github.com/stadium-software/samples-upgrading)