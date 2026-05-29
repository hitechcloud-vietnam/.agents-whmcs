# WHMCS Bootstrap Components

## Overview

WHMCS uses Bootstrap as its CSS framework for responsive design. Understanding Bootstrap components helps customize WHMCS templates effectively.

## Bootstrap Version

WHMCS v8+ uses Bootstrap 5. Check your version in Configuration > General Settings.

## Grid System

### Basic Grid

```smarty
<div class="container">
    <div class="row">
        <div class="col-md-8">
            Main content
        </div>
        <div class="col-md-4">
            Sidebar
        </div>
    </div>
</div>
```

### Responsive Columns

```smarty
<div class="row">
    <div class="col-12 col-md-4">
        {/* Mobile: full width, Desktop: 1/3 */}
    </div>
    <div class="col-12 col-md-4">
        {/* Mobile: full width, Desktop: 1/3 */}
    </div>
    <div class="col-12 col-md-4">
        {/* Mobile: full width, Desktop: 1/3 */}
    </div>
</div>
```

## Buttons

### Button Classes

```smarty
<button type="button" class="btn btn-primary">Primary</button>
<button type="button" class="btn btn-secondary">Secondary</button>
<button type="button" class="btn btn-success">Success</button>
<button type="button" class="btn btn-danger">Danger</button>
<button type="button" class="btn btn-warning">Warning</button>
<button type="button" class="btn btn-info">Info</button>
<button type="button" class="btn btn-link">Link</button>
```

### Button Sizes

```smarty
<button class="btn btn-primary btn-lg">Large</button>
<button class="btn btn-primary">Default</button>
<button class="btn btn-primary btn-sm">Small</button>
<button class="btn btn-primary btn-xs">Extra Small</button>
```

### Button States

```smarty
<button class="btn btn-primary" disabled>Disabled</button>
<button class="btn btn-primary active">Active</button>
<button class="btn btn-primary loading">Loading...</button>
```

## Forms

### Basic Form

```smarty
<form>
    <div class="form-group">
        <label for="email">Email address</label>
        <input type="email" class="form-control" id="email" 
               placeholder="Enter email">
    </div>
    <div class="form-group">
        <label for="password">Password</label>
        <input type="password" class="form-control" id="password" 
               placeholder="Password">
    </div>
    <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

### Inline Form

```smarty
<form class="form-inline">
    <div class="form-group">
        <label for="email">Email</label>
        <input type="email" class="form-control" id="email">
    </div>
    <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

### Form with Validation

```smarty
<form>
    <div class="form-group has-success">
        <label class="control-label" for="inputSuccess">
            Input with success
        </label>
        <input type="text" class="form-control form-control-success" 
               id="inputSuccess">
    </div>
    <div class="form-group has-error">
        <label class="control-label" for="inputError">
            Input with error
        </label>
        <input type="text" class="form-control form-control-error" 
               id="inputError">
    </div>
    <div class="form-group has-warning">
        <label class="control-label" for="inputWarning">
            Input with warning
        </label>
        <input type="text" class="form-control form-control-warning" 
               id="inputWarning">
    </div>
</form>
```

## Tables

### Basic Table

```smarty
<table class="table">
    <thead>
        <tr>
            <th>Column 1</th>
            <th>Column 2</th>
            <th>Column 3</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Data 1</td>
            <td>Data 2</td>
            <td>Data 3</td>
        </tr>
    </tbody>
</table>
```

### Table Variants

```smarty
<table class="table table-striped">     {* Striped rows *}
<table class="table table-bordered">     {* Bordered *}
<table class="table table-hover">        {* Hover rows *}
<table class="table table-condensed">   {* Condensed *}
<table class="table table-dark">         {* Dark variant *}
```

### Responsive Table

```smarty
<div class="table-responsive">
    <table class="table">
        {/* Table content */}
    </table>
</div>
```

## Panels

### Panel Component

```smarty
<div class="panel panel-default">
    <div class="panel-heading">
        <h3 class="panel-title">Panel Title</h3>
    </div>
    <div class="panel-body">
        Panel content
    </div>
    <div class="panel-footer">
        Panel footer
    </div>
</div>
```

### Panel Variants

```smarty
<div class="panel panel-primary">
<div class="panel panel-success">
<div class="panel panel-info">
<div class="panel panel-warning">
<div class="panel panel-danger">
```

## Alerts

### Alert Boxes

```smarty
<div class="alert alert-success">
    <strong>Success!</strong> Your action was successful.
</div>

<div class="alert alert-info">
    <strong>Info!</strong> This is an informational message.
</div>

<div class="alert alert-warning">
    <strong>Warning!</strong> Please review your information.
</div>

<div class="alert alert-danger">
    <strong>Error!</strong> Something went wrong.
</div>
```

### Dismissible Alert

```smarty
<div class="alert alert-warning alert-dismissible">
    <button type="button" class="close" data-dismiss="alert">
        &times;
    </button>
    <strong>Warning!</strong> This alert can be closed.
</div>
```

## Modals

### Basic Modal

```smarty
<!-- Trigger -->
<button type="button" class="btn btn-primary" 
        data-toggle="modal" data-target="#myModal">
    Open Modal
</button>

<!-- Modal -->
<div class="modal fade" id="myModal" tabindex="-1" role="dialog">
    <div class="modal-dialog" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <button type="button" class="close" 
                        data-dismiss="modal">
                    <span>&times;</span>
                </button>
                <h4 class="modal-title">Modal Title</h4>
            </div>
            <div class="modal-body">
                <p>Modal content...</p>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-default" 
                        data-dismiss="modal">Close</button>
                <button type="button" class="btn btn-primary">
                    Save
                </button>
            </div>
        </div>
    </div>
</div>
```

## Navigation

### Nav Tabs

```smarty
<ul class="nav nav-tabs">
    <li class="active">
        <a href="#home" data-toggle="tab">Home</a>
    </li>
    <li>
        <a href="#profile" data-toggle="tab">Profile</a>
    </li>
    <li>
        <a href="#messages" data-toggle="tab">Messages</a>
    </li>
</ul>

<div class="tab-content">
    <div class="tab-pane active" id="home">
        Home content
    </div>
    <div class="tab-pane" id="profile">
        Profile content
    </div>
    <div class="tab-pane" id="messages">
        Messages content
    </div>
</div>
```

### Pills

```smarty
<ul class="nav nav-pills">
    <li class="active">
        <a href="#">Home</a>
    </li>
    <li>
        <a href="#">Profile</a>
    </li>
</ul>
```

### Dropdown

```smarty
<ul class="nav nav-pills">
    <li class="dropdown">
        <a class="dropdown-toggle" data-toggle="dropdown" href="#">
            Dropdown <span class="caret"></span>
        </a>
        <ul class="dropdown-menu">
            <li><a href="#">Action</a></li>
            <li><a href="#">Another Action</a></li>
        </ul>
    </li>
</ul>
```

## Badges and Labels

### Badges

```smarty
<span class="badge">Default</span>
<span class="badge badge-primary">Primary</span>
<span class="badge badge-success">Success</span>
<span class="badge badge-info">Info</span>
<span class="badge badge-warning">Warning</span>
<span class="badge badge-danger">Danger</span>

<!-- In navbar -->
<ul class="nav nav-pills">
    <li><a href="#">Inbox <span class="badge">3</span></a></li>
</ul>
```

## Cards (Bootstrap 4+)

### Basic Card

```smarty
<div class="card">
    <div class="card-header">
        Card Header
    </div>
    <div class="card-body">
        <h5 class="card-title">Card Title</h5>
        <p class="card-text">Card content...</p>
        <a href="#" class="btn btn-primary">Button</a>
    </div>
    <div class="card-footer text-muted">
        Card Footer
    </div>
</div>
```

### Card with Image

```smarty
<div class="card" style="width: 18rem;">
    <img class="card-img-top" src="image.jpg" alt="Card image cap">
    <div class="card-body">
        <h5 class="card-title">Card Title</h5>
        <p class="card-text">Some text...</p>
        <a href="#" class="btn btn-primary">Go somewhere</a>
    </div>
</div>
```

## Progress Bars

```smarty
<div class="progress">
    <div class="progress-bar" role="progressbar" 
         style="width: 25%;" 
         aria-valuenow="25" 
         aria-valuemin="0" 
         aria-valuemax="100">
        25%
    </div>
</div>

<!-- Variants -->
<div class="progress-bar bg-success" style="width: 50%;"></div>
<div class="progress-bar bg-info" style="width: 25%;"></div>
<div class="progress-bar bg-warning" style="width: 75%;"></div>
<div class="progress-bar bg-danger" style="width: 100%;"></div>
```

## Utility Classes

### Spacing

```smarty
<div class="m-0">Margin 0</div>
<div class="mt-3">Margin top 3</div>
<div class="mb-3">Margin bottom 3</div>
<div class="ml-auto">Margin left auto</div>
<div class="p-3">Padding 3</div>
```

### Text

```smarty
<p class="text-left">Left aligned</p>
<p class="text-center">Center aligned</p>
<p class="text-right">Right aligned</p>
<p class="text-muted">Muted text</p>
<p class="text-primary">Primary text</p>
<p class="text-success">Success text</p>
```

### Colors

```smarty
<p class="text-primary">Primary</p>
<p class="text-success">Success</p>
<p class="text-info">Info</p>
<p class="text-warning">Warning</p>
<p class="text-danger">Danger</p>
<p class="bg-primary text-white">White on primary</p>
```

## See Also

- [Custom CSS](../whmcs-custom-css.md)
- [Font Awesome Icons](../whmcs-fontawesome-icons.md)
- [Client Area Templates](../whmcs-clientarea-templates.md)