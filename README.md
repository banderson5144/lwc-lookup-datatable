# lwc-lookup-datatable
This repo demonstrates using a Lookup component in a LWC datatable.

Since the Lookup data type is not natively supported by the [`lighting-datatable`](https://developer.salesforce.com/docs/component-library/bundle/lightning-datatable/example), I had to implement a [Custom Data Type](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.data_table_custom_types)

During my development there were a couple of challenges I had to overcome.

The first was passing a function to custom HTML template for my Custom Data Type. In the documentation from Salesforce, the `typeAttributes` is used to pass custom properties that can be used in your Custom Data Type:

```html
<template>
    <lightning-badge 
        label={typeAttributes.accountName}
        icon-name="standard:account">
    </lightning-badge>
</template>
```

However, my need was not to pass a simple property (I.e. String, Number, etc.). I needed to pass through a function so I can handle the `onsearch` event that is emitted from [Philippe Ozil's](https://github.com/pozil) [Lookup Component](https://github.com/pozil/sfdc-ui-lookup-lwc#handling-selection-changes-optional). I was really struggling on how to do this until I reached out to the [SFXD Discord community](https://discord.gg/sfxd). A user by the name [Suraj Pillai](https://github.com/surajp) provided me with a very elegant solution. To bind a function to a `typeAttribute` you can do this:

```javascript
//myCustomTypeDatatable.js
import LightningDatatable from 'lightning/datatable';
import customLookupEditTemplate from './customLookupEdit.html';

export default class MyCustomTypeDatatable extends LightningDatatable {
    static customTypes = {
        customLookupEdit: {
            template: customLookupEditTemplate,
            standardCellLayout: true,
            typeAttributes: ['onsearch'],
        }
    }
}
```

```html
<!-- customLookupEdit.html -->
<template>
    <c-lookup-v2 data-inputable="true"
                onsearch={typeAttributes.onsearch}>
    </c-lookup-v2>
</template>
```

```javascript
//parentComponent.js
import { LightningElement, track } from 'lwc';

export default class ParentComponent extends LightningElement {

    handleSearchFunction(event)
    {
        const lookupElement = event.target;
        lookupElement.setSearchResults('[{"title":"title","subtitle":"subtitle","sObjectType":"type","id":"0010R00000yvEyRQAU","icon":"icon"}]');
    }

    columns = { label: 'foo', fieldName: 'foo', type: 'customLookupEdit', typeAttributes: {'onseach':this.handleSearchFunction} };

}
```

Once I was able to implement this, I noticed that I was not triggering the `oncellchange` event. After doing a lot of debugging of Salesforce's internal `datatable.js` code, I noticed that @pozil Lookup component was missing two public API properties. They were the `validity` and `value` property. I currently have a [Pull Request](https://github.com/pozil/sfdc-ui-lookup-lwc/pull/131) in to add this:

```javascript
@api
get validity() {
    return {valid:!this._errors || this._errors.length === 0};
}

@api
get value() {
    return this.getSelection();
}
```

Once I added this to my local copy in my org, I was able to trigger the `oncellchange` event successfully. This allows the `draftValues` property to store the temporary changes in the table.