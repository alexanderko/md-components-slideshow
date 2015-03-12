class: center, middle, inverse

# AngularJS - Component development techniques

---

# Agenda
1. Configure components via `config` attribute
1. Extract specific logic from directive
1. Directive controllers

---
class: center, middle, inverse

# Configure components via `config` attribute

---
class: center, middle, inverse

# Table example

---
layout: true
template: default

### Configure components via config attribute - table

---

# Usage

```html
<th-grid 
    configuration="getTableConfig()" 
    source="source" 
    destination="data.destination"
    computed="computedTotal">
</th-grid>
```

---

# Configuration

```javascript
{
    selectable: true,
    sortingInfo: {
        key: 'last_name',
        reverse: false
    },
    fields: [
        {
            title: '"RESOURCES.TABLE.EMPLOYEES.COLUMNS.NAME" | translate',
            content: 'item.first_name + " " + item.last_name',
            key: 'last_name',
            subtotal: 'selection | formatSelectionInfo'
        },
        {
            title: '"RESOURCES.TABLE.EMPLOYEES.COLUMNS.ROLE" | translate',
            content: '"RESOURCES.TABLE.EMPLOYEES.ROLE."' + 
                ' +  item.role.toUpperCase() | translate',
            key: 'role'
        }
    ]
}
```

???

* использование объекта для конфигурации позволяет легко расширять его пока компонент развивается
* Content of each column can be configured thanks to use of `eval` function but it has performance drawbacks.

---
class: center, middle, inverse
layout: false

## Period picker example

---
layout: true
template: default

### Configure components via config attribute - period picker

---

# Usage

```html
<period-picker
    configuration="periodPickerConfig"
    period="period"
    daily-activity="dailyActivity"
    apply="applyPeriodFilter()">
</period-picker>
```

---

# Configuration

```javascript
$scope.periodPickerConfig = [
    periodPickerConfig.today(),
    periodPickerConfig.yesterday(),
    periodPickerConfig.current(),
    periodPickerConfig.previous(),
    periodPickerConfig.template({
        title: "'CUSTOM' | translate",
        templateUrl: 'views/directives/calendar_template.html'
    })
]
```

---

# Items construction

```javascript
function today() {
    return new Item('today').setRange(new DateRange.Current().day);
}

function yesterday() {
    return new Item('yesterday').setRange(new DateRange.Previous().day);
}

function current(subItems) {
    var rootItem = new Item("current");
    addSubItems(rootItem, new DateRange.Current(), subItems);
    return rootItem;
}

function template(config) {
    return new Template(config);
}
```

???

* каждый элемент является объектом с функциями для его настройки
* в конструктор - ключ для перевода
* в ренж - функция создания периода

---

### Item prototype

```javascript
function Item(config, dateRange) {
    // ...
}
Item.prototype = {
    activate: function () {},
    addSubItem: function (item) {},
    setRange: function (range) {},
    isRangeEqual: function (range) {},
    deactivate: function () {},
    detectActive: function (range) {}
}
function Template(config) {
    this.title = config.title;
    this.templateUrl = config.templateUrl;
}
Template.prototype.deactivate = function () {
    this.active = false;
}
```

???

* также элементы содержат вспомогательные функции для проверки состояния

---
layout: false
class: center, middle, inverse

# Extract specific logic from directive

---
class: center, middle, inverse

# Calendar and daily activity example

---
layout: true
template: default

### Calendar and daily activity example

---

# Usage

```html
<calendar
    period="period" 
    daily-activity="dailyActivity" 
    apply="apply()">
</calendar>
```

```javascript
$scope.dailyActivity = new DailyActivity(
    selectedProjectIds,
    selectedEmployeeIds,
    selectedCategoryIds
);
```

---

# Interface
```javascript
function DailyActivity(projectIds, employeeIds, categoryIds) {
    //...
}
DailyActivity.prototype = {
    fetch: function (period) {},
    hasActivity: function (date) {},
    equals: function (anotherDailyActivity) {}
}
```

???

* `DailyActivity` encapsulates routine and information required to get daily activity statuses.
    * get statuses for desired period using `promise`
    * check status of a date
    * compare with another instance of service. 

* This comparison makes it possible to simplify client code and avoid unnecessary requests.
* Use of `promise` makes it easy to test directive in isolation and add callbacks. 

Programmatic change state - chart legend.

---
class: center, middle, inverse
layout: false

# `&` scope attributes

???

Another way to extract functionality from directive is `&` scope attribute. We used it to pass data for chart tooltip.

---
layout: true
template: default

### & scope attributes

---

# Usage

```xml
<th-chart 
    source="data.destination" 
    tooltip-source="getTooltipSource(item, bar)" 
    configuration="getChartConfig()" 
    chart-lines="chartLines" 
    legend="legend">
</th-chart>
```

---

# Part of directive definition object and linking code

```javascript
//...
scope: {
    source: '=',
    tooltipSource: '&',
    configuration: '=',
    chartLines: '=',
    legend: '='
},
//...
link: function (scope) {
    function barHoverHandler(item, bar) {
        //...
        promise = scope.tooltipSource({item: item, bar: bar});
        promise.then(function (response) { /*  */ } );
        //...
    }
}
```

???

* Depending on current chart type `getTooltipSource(item, bar)` may query data for hovered item or read it from item itself.
* Function returns promise and it can be resolved right from place where it was created even before any callbacks added.

---
class: center, middle, inverse
layout: false

# Use directive controllers for communication

---
class: center, middle, inverse

# Vertical space occupation example

---
template: default
layout: true

### Directive controllers - vertical space occupation
---

# Usage

```xml
<div th-selection-manager>
    <th-selection selection="selection.projects"></th-selection>
    <th-selection selection="selection.employees"></th-selection>
    <th-selection selection="selection.categories"></th-selection>
</div>
```

???

Большенство атрибутов и обычный текст удалены

---

# th-selection-manager controller

```javascript
{
    controller: function ($scope, $element, $window) {
        this.addSelection = function (selection) {
            selections.push(selection);
        };

        this.removeSelection = function (selection) {
            var index = selections.indexOf(selection);
            selections.splice(index, 1);
        };

        this.selectionChanged = function () {
            // triggers updateLimits if container width is known
        }

        // code to trigger updateLimits after window resize
    }
}
```

???

* Менеджер состоит только из контроллера
* контроллер регистрирует компоненты

---

# th-selection-manager updateLimits

```javascript
function updateLimits() {
    var selectionsWithItems = [];
    angular.forEach(selections, function(selection) {
        selection.resetLimit();
        selectionsWithItems.push(selection);
    });
    while (selectionsWithItems.length) {
        for (var i = 0; i < selectionsWithItems.length;) {
            selection = selectionsWithItems[i];
            if (selection.hasItemsToDisplay()) {
                // ...
                if (isSpaceAvailable) {
                    selection.incrementLimit();
                    i++;
                    continue;
                }
            }
            selectionsWithItems.splice(i, 1);
        }
    }
    angular.forEach(selections, function(selection) {
        selection.digest();
    });
}
```

???

* Использование модели контроллером для распределения пространства

---

# th-selection - Selection model

```javascript
function Selection(scope, element) {
    this.hasItemsToDisplay = function () { },
    this.getNextItemWidth = function () { },
    this.resetLimit = function () { },
    this.incrementLimit = function () { },
    this.digest = function () {
        scope.$digest();
    }
}
```

???

* интерфейс модели используемой для разных типов сущностей
* digest - для обновления шаблона

---

# th-selection - directive

```javascript
{
    require: '?^thSelectionManager',
    link: function (scope, element, attr, thSelectionManager) {
        if (thSelectionManager) {
            var selection = new Selection(scope, element);
            thSelectionManager.addSelection(selection);
            scope.$on('$destroy', function() {
                thSelectionManager.removeSelection(selection);
            });
        } else {
            // use fake manager implementation
            var thSelectionManager = {
                selectionChanged: angular.noop
            };
        }

        // trigger thSelectionManager.selectionChanged() when selection changed
    }
}
```

???

* в линк фикции директива регистрирует свою модель в родительской директиве
* Удаление модели из списка необходимо для корректной работы после уничтожения скоупа кагда, к примеру, срабатывает ng-if
* использование контролеров для вывода ошибок валидации

---
layout: false
class: center, middle, inverse

# Thank you for your attention
