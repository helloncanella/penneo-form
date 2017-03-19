# Refactoring a beautiful form

Consider the form given by the [link](https://helloncanella.github.io/without-username/)

Beautiful, isn't ?

Its features are restricted to the exhibition of three inputs, validated when we press the submission button.

One of the ways to model it using React is shown here,

```javascript
import React, { Component } from 'react'
import './style.scss'

export default class Form extends Component {
    constructor() {
        super()
        this.state = {
            sentData: false
        }
        this.validateFields = this.validateFields.bind(this)
    }

    validateEmail(email) {
        var re = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
        return re.test(email);
    }

    validateFields(username, email, password, confirmationPassword) {

        if (!(username && email && password && confirmationPassword)) {
            this.setState({ error: 'one or more fields are empty' })
            return false;
        } else if (!this.validateEmail(email)) {
            this.setState({ error: 'email format is not correct' })
            return false;
        } else if (password !== confirmationPassword) {
            this.setState({ error: 'passwords don\'t match' })
            return false;
        }

        return true;
    }

    fetchData() {
        var data = []

        var elements = this.form.querySelectorAll('input:not(*[type="submit"])')

        elements.forEach(({ name, value, type, checked }) => {
            data[name] = value
        })

        return data
    }

    send(data){
        //send data algorithm
        this.setState({sentData: true})
    }

    onSubmit(e) {
        e.preventDefault()

        var data = this.fetchData()
        var email = data.email
        var password = data.password
        var confirmationPassword = data['password-confirmation']
        var username = data.username
       

        if (this.validateFields(username, email, password, confirmationPassword)) {
            this.send(data) 
        }
    }

    render() {
        return (
            this.state.sentData ?
                <h1 className="sent">Sent</h1>
                :
                <div className="form-wrapper">
                    <h1>A beautiful form</h1>
                    <form onSubmit={this.onSubmit.bind(this)} ref={e => this.form = e}>
                        <div className="content">
                            <h4 className="error">{this.state.error}</h4>
                            <input type="text" name="username" placeholder="username" />
                            <input type="text" name="email" placeholder="email"/>
                            <input type="password" name="password" placeholder="password" />
                            <input type="password" name="password-confirmation" placeholder="password confirmation" />
                            <input type="submit" />
                        </div>
                    </form>
                </div>
        )
    }
}
```


A patient and attentive reading of the code above inform us the component is responsible for the rendering of the inputs, validation of its data and the sending of it to somewhere. 

In fact, it works. However, to assign too many responsibilities to a single component can make the maintainability a difficult task.

Just the introduction of one single field and another validation rule require significant changes, violating the [Open/Closed principle]([https://en.wikipedia.org/wiki/Open/closed_principle).

The maintainability of this component can be improved if we share its responsibilities between other classes.

So, the task of display the inputs and expose its data can be restricted to the same Form component, that will receive by `props` the instruction to render its `fieldsets`.

```javascript
import React, { Component } from 'react'

class Form extends Component {
    constructor() {
        super()
        this.onSubmit = this.onSubmit.bind(this)
    }

    //public method
    getData() {
        return this.data
    }

    fetchData() {
        let data = []

        const fieldsets = this.form.querySelectorAll("fieldset")

        fieldsets.forEach(fieldset => {
            const { name } = fieldset

            data.push({ 
                name, 
                inputs: this.fetchInputsData(fieldset) 
            })
        })

        return data
    }

    fetchInputsData(fieldset) {
        var data = {}

        var inputs = fieldset.querySelectorAll('input:not(*[type="submit"])')
        inputs.forEach(({ name, value }) => data[name] = value)

        return data
    }

    renderFieldsets() {
        const { fieldsets } = this.props

        return fieldsets.map(({ name, inputs }, index) => {
            return <fieldset name={name} key={index}>{this.renderInputs(inputs)}</fieldset>
        })
    }

    renderInputs(inputs) {
        return inputs.map((inputProps, index) => {
            return <input {...inputProps} key={index} />
        })
    }

    onSubmit(e) {
        e.preventDefault()
        this.data = this.fetchData()
        this.props.onSubmit()
    }

    render() {
        return (
            <form onSubmit={this.onSubmit} ref={e => this.form = e}>
                <h1>A beautiful form</h1>
                <div className="content">
                    <h4 className="error">{this.props.validationError}</h4>
                    {this.renderFieldsets()}
                    <input type="submit" />
                </div>
            </form>
        )
    }
}

export default Form
```


The manipulation of the form data (validating and sending) will be done by a wrapper, with the help of external Classes.

```javascript
import React, { Component } from 'react'
import Form from './Form.jsx'
import fieldsets from './fieldsets.js'
import FieldsetsValidator from './fieldsets-validator.js'
import './style.scss'

class FormWrapper extends Component {
    constructor() {
        super()

        this.state = {
            dataWasSent: false,
            validationError: null
        }

        this.onSubmit = this.onSubmit.bind(this)
    }

    setError(error) {
        this.setState({ validationError: error })
    }

    validateFields(formData) {
        const validator = new FieldsetsValidator()

        try {
            validator.validate(formData)
        } catch (error) {
            this.setError(error)
            return false
        }

        return true;
    }

    sendData() {
        /***
            SENDING ALGORITHM!!!!!!!!!
        **/

        this.setState({ dataWasSent: true })
    }

    onSubmit() {
        const formData = this.form.getData();
        this.validateFields(formData) && this.sendData(formData)
    }

    formProps() {
        return {
            onSubmit: this.onSubmit,
            validationError: this.state.validationError,
            fieldsets,
            ref: e => this.form = e
        }
    }

    render() {
        const SentMessage = () => <h1 className="sent">Sent</h1>;
        return this.state.dataWasSent ? <SentMessage /> : <Form {...this.formProps() } />
    }
}

export default FormWrapper

```

As we can see below, the validation is done by an auxiliary class `FieldsetsValidator`, that verifies if one of the fields are empty and activates the validation of each one.

```javascript
import fieldsets from './fieldsets.js'

class FieldsetsValidator {

    verifyIfThereIsInputEmpty(fieldsetsData) {
        let inputsData = []

        fieldsetsData.forEach(({ inputs }) => {
            for (let name in inputs) {
                if (!inputs[name]) throw 'one or more fields are empty'
            }
        })
    }

    validateEachField(data) {
        fieldsets.forEach(({ name, validator: validate }) => {
            const inputsData = data.filter(item => item.name === name)[0].inputs
            if (validate) validate(inputsData)
        })
    }

    validate(data) {
        this.verifyIfThereIsInputEmpty(data)
        this.validateEachField(data)
    }
}

export default FieldsetsValidator
```


The instructions to render and validate the inputs is described here.

```javascript
const validateEmail = ({ email }) => {
    const regex = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    if (!regex.test(email)) throw 'email format is not correct!'
}

const validatePasswords = ({ password, confirmation }) => {
    if (password !== confirmation) throw 'passwords do not match!'
}

const validateUsername = ({ username }) => {
    if (!isNaN(username)) { throw 'username cannot be a number' }
}


const fieldsets = [
    {
        name: "email",
        inputs: [{ name: 'email', type: 'text', placeholder: "email" }],
        validator: validateEmail
    },

    {
        name: "passwords",
        validator: validatePasswords,
        inputs: [
            { name: 'password', type: 'password', placeholder: "password" },
            { name: 'confirmation', type: 'password', placeholder: "password confirmation" }
        ],
    },

]

export default fieldsets
```

In the future, for example, we can insert a username field and require its value cannot be numerical. With a simple insertion in `fieldsets`, we can do it. 

```javascript
const fieldsets = [
    {
        name: "username",
        inputs: [{ name: 'username', type: 'text', placeholder: "username" }],
        validator: validateUsername
    }

    {
        name: "email",
        inputs: [{ name: 'email', type: 'text', placeholder: "email" }],
        validator: validateEmail
    },

    {
        name: "passwords",
        validator: validatePasswords,
        inputs: [
            { name: 'password', type: 'password', placeholder: "password" },
            { name: 'confirmation', type: 'password', placeholder: "password confirmation" }
        ],
    }
]
```


The result is available [here](https://helloncanella.github.io/with-username/)


In fact, the solution presented is more verbose. However, futures modifications, such the insertion of more fields and validation rules can add more complexity, making the component more prone to errors. The solution adopted avoids theses risks, at the same time it improves its readability.  





