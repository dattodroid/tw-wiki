---
tags:
  - snippet
  - js
---

# Service Script snippets

## External Resources

- [Unofficial- ThingWorx Snippet Enhanced Documentation](https://community.ptc.com/t5/IoT-Tips/Unofficial-ThingWorx-Snippet-Enhanced-Documentation/ta-p/818733){:target="\_blank"}

#### Create an InfoTable from JSON

```js
let my_it_json = {
    dataShape: {
        fieldDefinitions: {
            'name': {
                name: 'name',
                baseType: 'STRING',
                ordinal: 0
            },
            'description': {
                name: 'description',
                baseType: 'STRING',
                ordinal: 1
            },
        }
    },
    rows: []
};

let my_it = Resources["InfoTableFunctions"].FromJSON({
    json: it_json
});
```
