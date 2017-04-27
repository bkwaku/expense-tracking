# A mock expense tracking app

We are building a small application from scratch to keep track of our expenses. 

Each `Record` will consist of a `date`, a `title` and an `amount`. A record will be treated as `Credit` if its amount is greater than zero, otherwise it will be treated as `Debit`.

Initializing the project:
```bash
rails new expense-tracking

git init
git add -A
git commit -m "first commit"
git remote add origin https://github.com/joshuaai/expense-tracking.git
git push -u origin master
```

Add the react gem to the `Gemfile`:
```ruby
# Bootstrap
gem 'bootstrap-sass'
# React 
gem 'react-rails', '~> 1.0'
```

In `custom.scss` import the bootstrap-sass gem dependencies:
```css
@import "bootstrap-sprockets";
@import "bootstrap";
```

Install the gems:
```bash
bundle install
```

Run the react installer which will create a `components.js` file and a `components` directory under `app/assets/javascripts` where our React components will live.
```bash
rails g react:install
```

## Creating the Record Resource
We are going to build a Record resource, which will include a `date`, a `title`, and an `amount`. Instead of using the `scaffold` generator, we are going to use the `resource` generator, as we are not going to be using all of the files and methods created by the scaffold generator. 

```bash
rails g resource Record title date:date amount:float

rails db:create db:migrate
```

We can create a couple of records through the `rails console`:
```bash
Record.create title: 'Record 1', date: Date.today, amount: 500
Record.create title: 'Record 2', date: Date.today, amount: -100
```

## Nesting Components: Listing Records
Create the index action in `records_controller.rb` that renders any existing record:
```ruby
class RecordsController < ApplicationController
    def index
        @records = Record.all
    end
end
``` 

Create an `index.html.erb` file under `apps/views/records` that will act as a bridge between our Rails app and our React components. This will use the `react_component` helper method, which receives the name of the React component we want to render along with the data we want to pass into it.

```html
<%= react_component 'Records', { data: @records } %>
```

We create our first component, inside the `app/assets/javascripts/components` path, to be the BaseComponent -> `base_component.js.jsx`. We will use ES6 for this:

```jsx
class BaseComponent extends React.Component {
    _bind(...methods) {
        methods.forEach((method) => this[method] = this[method].bind(this));
    }
}
```

We now create the `records.js.jsx` component inside the same path as above. React components rely on the use of `properties` to communicate with other components and `states` to detect whether a re-render is required or not. We need to initialize our component's state and properties with the desired values. We can now create our first 

```jsx
'use strict';

class Records extends BaseComponent {
    constructor(props) {
        super();
        this._bind();
        this.state = { records: props.data }
    }

    render() {
        return (
            <div className="records">
                <h2 className="title"> Records </h2>
                <div className="row"></div>
            </div>
        );
    }

}

Records.defaultProps = {
    records: = []
}
```

The `defaultProps` method will initialize our component's properties incase we forget to send any data when instantiating it, and `this.state` in the constructor will generate the initial state of our component.

We now implement a simple string formatter helper method and make it accesible to all of our jsx files. Create a new `utils.coffee` file under javascripts/ with the following contents:

```coffee
@amountFormat = (amount) ->
    '$ ' + Number(amount).toLocaleString()
```

We need to create a new `Record` component to display each individual record. Create a new file `record.js.jsx` under the `javascripts/components` directory. The `Record` component will display a table row containing table cells for each record attribute. 

```jsx
'use strict';

class Record extends BaseComponent {
    constructor(props) {
        super();
        this.state = { title: this.props.title, date: this.props.date, amount: 0 }
    }

    render() {
        return (
            <tr>
                <td><input className="form-control" type="text" defaultValue={this.props.record.date} ref="date" /></td>
                <td><input className="form-control" type="text" defaultValue={this.props.record.title} ref="title" /></td>
                <td><input className="form-control" type="number" defaultValue={this.props.record.amount} ref="amount" /></td>
            </tr>
        );
    }
}

Record.defaultProps = {
    title: 'title', date: 'date', amount: 0
}
```

## Parent-Child Communication: Creating Records
Now that we are displaying all the existing records, we can include a form to create new records. 

First, we need to add the `create` method to our Rails controller which will use *strong params*. In the `records_controller.rb` file, add:

```ruby
def create
    @record = Record.new(record_params)

    if @record.save
        render json: @record
    else
        render json: @record.errors, status: :unprocessable_entity
    end
end

private

    def record_params
        params.require(:record).permit(:title, :amount, :date)
    end
end
```

Next, we need to build a React component to handle the creation of new records. The component will have its own `state` to store `date`, `title` and `amount`. Create a new `record_form.js.jsx` file under javascripts/components with the following code:

```jsx
'use strict';

var initialState = {
    title: '', date: '', amount: ''
};

constructor(props) {
    super(props);
    this._bind('handleChange', 'valid', 'submit');
    this.state = initialState;
}

handleChange(event) {
    // Using string interpolation to dynamically define object keys, equivalent to @setState 
    // title: e.target.value when name equals title.
    var name = event.target.name;
    var obj = {};
    obj[""+name] = event.target.value;
    // setState() updates the component's state and schedules a UI verification/refresh based on the new state
    this.setState(obj);
}

valid() {
    return this.state.title && this.state.date && this.state.amount;
}

handleSubmit(event) {
    event.preventDefault();
    $.post('', {record: this.state}, (data) => {
        this.props.handleNewRecord(data);
        this.setState(initialState);
    });
}

render() {
    return (
        <form className="form-inline" onSubmit={this.handleSubmit}>
            <div className="form-group">
                <input type="text" className="form-control" placeholder="Date"
                        name="date" value={this.state.date} onChange={this.handleChange} />
            </div>
            <div className="form-group">
                <input type="text" className="form-control" placeholder="Title"
                        name="title" value={this.state.title} onChange={this.handleChange} />
            </div>
            <div className="form-group">
                <input type="number" className="form-control" placeholder="Amount"
                        name="amount" value={this.state.amount} onChange={this.handleChange} />
            </div>
            <button type="submit" className="btn btn-primary" disabled={!this.valid()}>
                Create Record
            </button>
        </form>
    );
}

RecordForm.propTypes = {
    handleNewRecord: React.PropTypes.func.isRequired
};
```

### Refactor: State Helpers
For a complex application with a multi-level JSON state, React includes some fancy state helpers to help you with some of the heavy lifting, no matter how deep your state is.

Before using these helpers, first we need to configure our Rails application to include them. In the `config/application.rb` file add `config.react.addons = true` at the bottom of the Application block:
```ruby
# config/application.rb

...
module Accounts
    class Application < Rails::Application
        ...
        config.react.addons = true
    end
end
```

Now we have access to the state helpers through `React.addons.update`, which will process our state object (or any other object we send to it) and apply the provided commands. The two commands we will be using are `$push` and `$splice`in `records.js.jsx`.

Components communicate with other components through properties (or @props). Our current component sends data back to the parent component through `this.props.handleNewRecord` to notify it about the existence of a new record.

Add the new `addRecord` method inside the `records.js.jsx` and create the new `RecordForm` element inside the `render()` method just after the `h2`.

```js
addRecord(record) {
    var records = React.addons.update(this.state.records, {$push: [record]});
    this.setState({ record: records});
}

render() {
    <div className="records">
        <h2 className="title"> Records </h2>
        <div className="row"></div>
        <RecordForm handleNewRecord={this.addRecord} />
    </div>
}
```

## Reusable Components: Amount Indicators
Let's add some boxes at the top of our window with some useful information. The goal for this section is to show 3 values: Total credit amount, total debit amount and Balance.

We can build a new AmountBox component which will receive three properties: amount, text and type. Create a new file called `javascripts/components/amount_box.js.jsx` and add:
```js
'use strict';

class AmountBox extends BaseComponent {
    render() {
        var panel = " panel panel-" + this.props.type;
        return (
            <div className="col-md-4">
                <div className={ panel }>
                    <div className="panel-heading">
                        { this.props.text }
                    </div>
                    <div className="panel-body">
                        {amountFormat(this.props.amount)}
                    </div>
                </div>
            </div>
        );
    }
}

AmountBox.propTypes = {
    type: React.PropTypes.string.isRequired,
    text: React.PropTypes.string.isRequired,
    amount: React.PropTypes.number.isRequired
};
```

We are just using Bootstrap's `panel` element to display the information in a "blocky" way, and setting the color through the `type` property.

Lets build the caluculator by adding the methods in the `Records` components that will perform the calculate actions:
```js
credits() {
    return this.state.records
        .filter( (record) => {
            return record.amount >= 0;
        })
        .reduce( (previous, current) => {
            return previous + parseFloat(current.amount)
        }, 0);
}

debits() {
    return this.state.records
        .filter( (record) => {
            return record.amount < 0;
        })
        .reduce( (previous, current) => {
            return previous + parseInt(current.amount)
        }, 0);
}

balance() {
    return this.debits() + this.credits();
}
```

Now that we have the calculator methods in place, we just need to create the `AmountBox` elements inside the `render()` method (just above the RecordForm component):
```js
return (
    <div className="records">
        <h2 className="title"> Records </h2>
        <div className="row">
            <AmountBox type="success" amount={this.debits()} text="Credit"/>
            <AmountBox type="danger" amount={this.credits()} text="Debit"/>
            <AmountBox type="info" amount={this.balance()} text="Balance"/>
        </div>
    </div>
);
```

## setState/replaceState: Deleting Records
We need a new `Actions` column in our records table, this column will have a Delete button for each record, pretty standard UI.

Add the `destroy` action to the `records-controller.rb` file:
```ruby
def destroy
    @record = Record.find(params[:id])
    @record.destroy
    head :no_content
end
```

In the Records component in `records.js.jsx`, and add the `Actions` column at the rightmost position of the table header:
```js
<div className="records">
    .
    .
    .
    <hr />
    <table className="table table-bordered">
        <thead>
            <tr>
                <th> Date </th>
                <th> Title </th>
                <th> Amount </th>
                <th> Actions </th>
            </tr>
        </thead>
        <tbody>
        {records}
        </tbody>
    </table>
</div> 
```

In the `Record` component in `record.js.jsx`, add an extra column with a Delete link:
```js
recordRow() {
    return (
        <tr>
            <td>{this.props.record.date}</td>
            <td>{this.props.record.title}</td>
            <td>{amountFormat(this.props.record.amount)}</td>
            <td>
                <a className="btn btn-danger" onClick={this.handleDelete}>Delete</a>
            </td>
        </tr>
    );
}
```

Let's add some functionality to it. As we learned from our RecordForm component, the way to go here is:
* Detect an event inside the child Record component (`onClick`).
* Perform an action (send a DELETE request to the server in this case).
* Notify the parent Records component about this action (sending/receiving a handler method through props)
* Update the Record component's state

For the first, re-open the `Record` component, `record.js.jsx` and add a new `handleDelete` method. The `onClick` attribute in the code above already sets the delete button to this method. The code is as follows:
```js
handleDelete(event) {
    var id = "records/" + this.props.record.id;
    event.preventDefault();
    $.ajax({
        method: 'DELETE',
        url: id,
        dataType: 'JSON',
        success: ( () => {
            this.props.handleDeleteRecord(this.props.record);
        })
    });
}
```

When the delete button gets clicked, `handleDelete` sends an AJAX request to the server to delete the record in the backend and, after this, it notifies the parent component about this action through the `handleDeleteRecord` handler available through props, this means we need to adjust the creation of `Record` elements in the parent component to include the extra property `handleDeleteRecord` in its `propTypes`, and also implement the actual handler method in the parent.

Add the `deleteRecord` methos in `records.js.jsx`:
```js
deleteRecord(record) {
    var index = this.state.records.indexOf(record);
    var records = React.addons.update(this.state.records, {$splice: [[index, 1]]});
    this.replaceState({ records: records });
}
```

Basically, our deleteRecord method copies the current component's records state, performs an index search of the record to be deleted, splices it from the array and updates the component's state, pretty standard JavaScript operations.

We introduced a new way of interacting with the state, replaceState; the main difference between `setState` and `replaceState` is that the first one will only update one key of the state object, the second one will completely override the current state of the component with whatever new object we send.

## Reactive Data Flow: Editing Records
We are adding an extra Edit button, next to each Delete button in our records table. When this Edit button gets clicked, it will toggle the entire row from a read-only state to an editable state, revealing an inline form where the user can update the record's content. 

After submitting the updated content or canceling the action, the record's row will return to its original read-only state.

We need to handle mutable data to toggle each record's state inside of our `Record` component. This is a use case of what React calls reactive data flow. Let's add an `edit` flag and a `handleToggle` method to `record.js.jsx`:

The `edit` flag will default to `false`, and `handleToggle` will change `edit` from `false` to `true` and vice versa, we just need to trigger `handleToggle` from a user `onClick` event.

The implementation of `recordForm` will follow a similar structure, but with input fields in each cell. We are going to use a new `ref` attribute for our inputs to make them accessible;
```js
constructor(props) {
    super();
    this._bind('handleDelete', 'handleToggle', 'recordRow', 'recordForm', 'handleEdit');
    this.state = {
        edit: false
    };
}

handleToggle(event) {
    event.preventDefault();
    this.setState({ edit: !this.state.edit} )
}

recordRow() {
    return (
        <tr>
            <td>{this.props.record.date}</td>
            <td>{this.props.record.title}</td>
            <td>{amountFormat(this.props.record.amount)}</td>
            <td>
                <a className="btn btn-default" onClick={this.handleEdit} > Update </a>
                <a className="btn btn-danger" onClick={this.handleDelete} > Delete </a>
            </td>
        </tr>
    );
}

recordForm() {
    return (
        <tr>
            <td><input className="form-control" type="text" defaultValue={this.props.record.date} ref="date" /></td>
            <td><input className="form-control" type="text" defaultValue={this.props.record.title} ref="title" /></td>
            <td><input className="form-control" type="number" defaultValue={this.props.record.amount} ref="amount" /></td>
            <td>
                <a className="btn btn-default" onClick={this.handleEdit} > Update </a>
                <a className="btn btn-danger" onClick={this.handleToggle} > Cancel </a>
            </td>
        </tr>
    );
}

render() {
    return this.state.edit ? this.recordForm() : this.recordRow();
}
```

To handle record updates, we need to add the `update` method to our the `records_controller.rb` file:
```ruby
def update
    @record = Record.find(params[:id])
    if @record.update(record_params)
        render json: @record
    else
        render json: @record.errors, status: :unprocessable_entity
    end
end
```

Back to our `record.js.jsx` component, we need to implement the `handleEdit` method which will send an AJAX request to the server with the updated record information.

It will notify the parent component by sending the updated version of the record via the `handleEditRecord` method, this method will be received through `@props`, the same way we did it before when deleting records:
```js
handleEdit(event) {
    event.preventDefault();
    var id = "records/" + this.props.record.id;
    var data = {
        title: ReactDOM.findDOMNode(this.refs.title).value,
        date: ReactDOM.findDOMNode(this.refs.date).value,
        amount: ReactDOM.findDOMNode(this.refs.amount).value
    };
    $.ajax({
        method: 'PUT',
        url: id,
        dataType: 'JSON',
        data: { record: data },
        success: ( (data) => {
            this.setState({ edit: false });
            this.props.handleEditRecord(this.props.record, data);
        })
    });
}
```
For the sake of simplicity, we are not validating user data, we just read it through `ReactDOM.findDOMNode(@refs.fieldName).value` and sending it verbatim to the backend. Updating the state to toggle edit mode on `success` is not mandatory, but the user will definitely thank us for that.

Add the `updateRecord` method to the `records.js.jsx` component:
```js
updateRecord(record, data) {
    var index = this.state.records.indexOf(record);
    var records = React.addons.update(this.state.records, { $splice: [[index, 1, data]] });
    this.replaceState({ records: records });
}
```