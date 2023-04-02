# React

## 1. React Basics

### Component Driven Design (CDD)

Bir web sitesi, tekrar eden component'lerin bir araya gelmesiyle oluşur. Örneğin, bir butonu ihtiyaç duyulan yerde kullanırken kullanılan yerde her seferinde tenımlamak yerine bir kere tanımlayıp bu buton component'ini ihtiyaç duyulan yerde kullanma prensibi denilebilir.

Bu yaklaşımın faydaları:

- Hızlı geliştirme
- Daha kolay maintenance: Component'te güncelleme veya genişleme yapılmak istendiğinde web sitesinin bir çok yerinde değişiklik yapmak yerine component'in ilgili dosyasında değişiklik yapmak yeterli olur.
- Reusability (DRY): Component'i tekrar tekrar yazmayı engeller.
- Standardization
- Modülerlik (SoC): Component'ler birbirinden etkilenmeden, bağımsız olarak tanımlanır.
- Test edilebilirlik: Sistemin her parçasının sorumluluklarını anlamak ve ayırmak daha kolay olduğu için daha büyük sistemler daha kolay test edilebilir.

#### Atomic Design

<img src="https://blogger.googleusercontent.com/img/a/AVvXsEhE756n2Pq5tp64UFIPrypg-FOD73tGID4dJoSaat-VjmghRhMFPwktrXAOalc9jKOLA_ih0dE51eqOjtihqmDsJR3PSbEfCjR1lOple3f1P7PsOhY0izytFNasGdWEs_YfBXZ05PFm6nK5P2ra66wHv2YLt065OSvDke58oanAW9D4eCznCOnm1AZKhw">

#### Imperative Approach vs Declarative Approach

Her etkileşimde istenen UI durumuna ulaşmak için adım adım DOM mutation'ı yapılır.

<img height="250" src="https://blogger.googleusercontent.com/img/a/AVvXsEhHkfquge781tdpdwxq3i01ecLt2UTSh75PZDgnNvvFHpnspVE_vy5JRMAKyUmx52oQicLG-DOxi0xl5zmFBxDFNMOH4diDUXhdAblg6i4vz7cT8NzpK6qlt2gre2m7RKcMPfDGeZIhZjOlMQPYqF2DxrbKV9JIqdVEzOUYLRynXWzZ2yDu4rba-j0EkQ">



React'te, istenen UI durumuna ulaşmak için adım adım talimatlar çalıştırılmaz. Bunun yerine, her sahne için istenen son UI tanımlanır.

<img height="250" src="https://blogger.googleusercontent.com/img/a/AVvXsEjdJzfpvdi6R69xsFa9uMIhHOMhAGjT_3Nr8wMug6LGMHXvfyAKMzkYgV-T3cCkZ6_o2mJngsMhrD8zU_2Ta52jwRqO2mnFEXegMfaYWNXtBNIOKFVEnXvki8Ih0iT4AZOUhHc95xY_KUoe6pMHxFM-NnGNK6k5XNbIyQPKBQoQoazMkYIGv3kdPxtPlA"><img height="250" src="https://blogger.googleusercontent.com/img/a/AVvXsEi5QypJ064IIg3UojuUt8U0AYXi0gAWy6ZSIms-aKNxYLsdqUyCyscTFsixx6HYkCxs1zI1sQ--86U1x8aDvVmww84d5HIdOimdaKQsg56DFx3NXMm6BydP9UK7kX34YvVgvW-CqUKwxh9gy4jBcCl8zW62fXfQ-bnxW3wpsRfe1M9QBKTEf492t7IBEw">



### How React Works?

DOM ağacında yapılacak herhangi bir değişiklik layout ve repaint'i tetikleyeceği için oldukça pahalıdır. Normal yapıda, element'te yapılacak ufak bir değişlik element'in tüm container'ını değiştiriyor. [[bkz.]](###How the browser renders a web page?)

React'in yaptığıysa, DOM ağacının aynısını Virtual DOM ağacı olarak tutar. Node'da herhangi bir değişiklik olduğunda Virtual DOM ağacında günceller. Ardından DOM ağacı ile karşılaştırıp, DOM ağacında sadece gerekli yerde değişikliğin yapılmasını sağlar. React bunu yaparken declarative yaklaşımı benimsemiştir. Component'leri ayrı ayrı tanımlayıp ihtiyaç duyulan yerde kullanarak esnek, tekrar kullanılabilir, test edilebilir ve hata ayıklaması kolay bir yapısı vardır.

React'in  JSX kodu aslında gerçek kod değildir. Browser'ın console'undan web sayfasının son haline bakıldığında custom component'lerin olmadığı, dönüştürülmüş hali görülebilir.



### Building a First Custom Component

Eklenecek component'ler için `src` klasörünün altına `components` adında klasör açılır. Dosya adı pascal case [5] kuralına uyacak şekilde component oluşturulur.

````jsx
// scr\components\Expenses.js
import './Expenses.css';
import { ExpenseItem } from './ExpenseItem.js';

const Expenses = (props) => {
  return (
      <div>
          <ExpenseItem className="red">
              <div>Item Description</div>
          </ExpenseItem>
      </div>
  );
}

export default ExpenseItem;


// scr\components\ExpenseItem.js
import './ExpenseItem.css';

const ExpenseItem = (props) => {
  return (
      <div>
          <h2 className={'expense-item ' + props.className}>
              Expense Item!
          </h2>
          {props.children} // children özel bir prop'tur.
      </div>
  );
}

export default ExpenseItem;
````



## 2. React State & Working with Events

### Working with Multiple States

````jsx
import './ExpenseForm.css';
import { useState } from "react";

const ExpenseForm = () => {
  //const [enteredTitle, setEnteredTitle] = useState('');
  //const [enteredAmount, setEnteredAmount] = useState('');
  //const [enteredDate, setEnteredDate] = useState('');
  const [userInput, setUserInput] = useState({
    enteredTitle: '',
    enteredAmount: '',
    enteredDate: '',
  });

  const titleChangeHandler = (event) => {
    //setEnteredTitle(event.target.value);
    setUserInput((prevState) => {
      return { ...prevState, enteredTitle: event.target.value };
    });
  };

  const amountChangeHandler = (event) => {
    //setEnteredAmount(event.target.value);
    setUserInput((prevState) => {
      return { ...prevState, enteredAmount: event.target.value };
    });
  };

  const dateChangeHandler = (event) => {
    //setEnteredDate(event.target.value);
    setUserInput((prevState) => {
      return { ...prevState, enteredDate: event.target.value };
    });
  };

  return (
    <form>
      <div className="new-expense__controls">
        <div className="new-expense__control">
          <label>Title</label>
          <input
            type="text"
            onChange={titleChangeHandler}
          />
        </div>
        <div className="new-expense__control">
          <label>Amount</label>
          <input
            type="number"
            min="0"
            step="0.01"
            onChange={amountChangeHandler}
          />
        </div>
        <div className="new-expense__control">
          <label>Date</label>
          <input
            type="date"
            min="2022-01-01"
            max="2025-12-31"
            onChange={dateChangeHandler}
          />
        </div>
      </div>
      <div className="new-expense__actions">
        <button type="submit">Add Expense</button>
      </div>
    </form>
  );
};

export default ExpenseForm;
````



### Child-to-Parent Component Communication (Bottom-up)

````jsx
// src\components\Expenses\ExpensesFilter.js
import './ExpensesFilter.css';

const ExpensesFilter = (props) => {
  const dropdownChangeHandler = (event) => {
    props.onChangeFilter(event.target.value); // Parent'ındaki (Expenses) filteredYear
      // değerinin güncellenmesi için yine o değeri değiştiren parent'ından gelen metot çağırılıyor.
  };

  return (
    <div className="expenses-filter">
      <div className="expenses-filter__control">
        <label>Filter by year</label>
        <select value={props.selected} onChange={dropdownChangeHandler}>
          <option value='2025'>2025</option>
          <option value='2024'>2024</option>
          <option value='2023'>2023</option>
          <option value='2022'>2022</option>
        </select>
      </div>
    </div>
  );
};

export default ExpensesFilter;


// src\components\Expenses\Expenses.js
import { useState } from "react";
import './Expenses.css';
import ExpenseItem from "./ExpenseItem";
import Card from "../UI/Card";
import ExpensesFilter from "./ExpensesFilter";

const Expenses = (props) => {
  const [filteredYear, setFilteredYear] = useState('2023');

  const filterChangeHandler = (selectedYear) => { // Child'ında (ExpensesFilter) bu metot
      // çağırılarak filteredYear değeri güncelleniyor.
    setFilteredYear(selectedYear);
  };

  return (
    <Card className="expenses">
      <ExpensesFilter selected={filteredYear} onChangeFilter={filterChangeHandler}/>
      <ExpenseItem
        title={props.items[0].title}
        date={props.items[0].date}
        amount={props.items[0].amount}
      />
    </Card>
  );
}

export default Expenses;
````



### Working with Refs (useRef Hook)

Değişken bir değerin değeri güncellendikten sonra güncellenen değeri re-render'a sebep olmadan saklamak için kullanılır.

DOM ağacındaki element'e doğrudan erişmek için kullanılır.

Bazı durumlarda useState'e alternatif olarak da kullanılabilir.

````jsx
const AddUser = (props) => {
  const usernameRef = useRef();
  const ageRef = useRef();
  //const [username, setUsername] = useState('');
  //const [age, setAge] = useState('');
  const [error, setError] = useState();

  const submitHandler = (event) => {
    event.preventDefault();
      
    const username = usernameRef.current.value;
    const age = ageRef.current.value;

    if (username.trim().length === 0 || age.trim().length === 0) {
      setError({
        title: 'Geçersiz Değer',
        message: 'Lütfen kullanıcı ismi ve yaş giriniz.',
      });
      return;
    }

    if (+age < 1) {
      setError({
        title: 'Geçersiz Yaş',
        message: 'Lütfen geçerli bir yaş giriniz.',
      });
      return;
    }

    const user = {
      id: Math.random(),
      username: username,
      age: +age,
    };

    props.onAddUser(user);
    usernameRef.current.value = '';
    ageRef.current.value = '';
    //setUsername('');
    //setAge('');
  }
  
  //const usernameChangeHandler = (event) => {
  //  setUsername(event.target.value);
  //};
  
  //const ageChangeHandler = (event) => {
  //  setAge(event.target.value);
  //};

  const errorHandler = () => {
    setError(null);
  };

  return (
    <div>
      {error && (
        <ErrorModal
          title={error.title}
          message={error.message}
          onConfirm={errorHandler}
        />
      )}
      <Card className={styles.input}>
        <form onSubmit={submitHandler}>
          <label htmlFor="username">Kullanıcı İsmi</label>
          <input
            id="username"
            type="text"
            ref={usernameRef}
            //value={username}
            //onChange={usernameChangeHandler}
          />
          <label htmlFor="age">Yaş</label>
          <input
            id="age"
            type="number"
            ref={ageRef}
            //value={age}
            //onChange={ageChangeHandler}
          />
          <Button type="submit">Kullanıcı Ekle</Button>
        </form>
      </Card>
    </div>
  );
};

export default AddUser;
````



### Handling Side Effects, Using Reducers & Context API

#### useEffect

Component oluştuktan sonra bir şeylerin yapılması gerektiği belirtilir. React, DOM güncellemelerini yaptıktan sonra onu çağıracaktır. Varsayılan olarak, hem ilk oluşturmadan sonra hem de her güncellemeden sonra çalışır.

Timeouts, subscriptions, event listeners gibi effect'lerin memory leak'e neden olmaması için temizlenmesi gerekir.

````jsx
//  Api'den gelen verilerle ülke, şehir seçimi örneği eklenecek.
````

#### useReducer

Kullanımı ve kullanım amacı useState'e benzemektedir. Kompleks ve custom state logic oluşturmayı sağlar.

````jsx
const [state, dispatchFn] = useReducer(reducer, initialState, <lazyInitFn>);
````

`reducer` fonksiyonu custom state logic'i tanımlar. `initialState` ilk değeri tanımlar ama bunun object olarak tanımlanması gerekir.

`useReducer`, geriye son `state`'i ve `dispatch` metodunu döndürür.

````jsx
import React, { useEffect, useReducer, useState } from 'react';
import Card from '../UI/Card/Card';
import classes from './Login.module.css';
import Button from '../UI/Button/Button';

const emailReducer = (state, action) => {
  if (action.type === 'USER_INPUT') {
    return { value: action.val, isValid: action.val.includes('@') };
  }
  return { value: '', isValid: false };
};

const Login = (props) => {
  const [enteredPassword, setEnteredPassword] = useState('');
  const [passwordIsValid, setPasswordIsValid] = useState();
  const [formIsValid, setFormIsValid] = useState(false);

  const [emailState, dispatchEmail] = useReducer(emailReducer, { value: '', isValid: undefined });

  useEffect(() => {
    setFormIsValid(
      emailState.isValid && enteredPassword.trim().length > 6
    );
  }, [emailState.isValid, enteredPassword]);

  const emailChangeHandler = (event) => {
    dispatchEmail({ type: 'USER_INPUT', val: event.target.value });
  };

  const passwordChangeHandler = (event) => {
    setEnteredPassword(event.target.value);
  };

  const validatePasswordHandler = () => {
    setPasswordIsValid(enteredPassword.trim().length > 6);
  };

  const submitHandler = (event) => {
    event.preventDefault();
    props.onLogin(emailState.value, enteredPassword);
  };

  return (
    <Card className={classes.login}>
      <form onSubmit={submitHandler}>
        <div
          className={`${classes.control} ${
            emailState.isValid === false ? classes.invalid : ''
          }`}
        >
          <label htmlFor="email">E-Posta</label>
          <input
            type="email"
            id="email"
            value={emailState.value}
            onChange={emailChangeHandler}
          />
        </div>
        <div
          className={`${classes.control} ${
            passwordIsValid === false ? classes.invalid : ''
          }`}
        >
          <label htmlFor="password">Şifre</label>
          <input
            type="password"
            id="password"
            value={enteredPassword}
            onChange={passwordChangeHandler}
            onBlur={validatePasswordHandler}
          />
        </div>
        <div className={classes.actions}>
          <Button type="submit" className={classes.btn} disabled={!formIsValid}>
            Login
          </Button>
        </div>
      </form>
    </Card>
  );
};

export default Login;
````

#### useContext

`useState` kullanarak bir değerin component'ler arasında taşınmasının zorlaştığı durumlarda, global state olarak kullanılır.

`useContext`‘i çağıran bir component, context değeri her değiştiğinde re-render edilecektir. (Provider-Consumer)

State'in kısa süreli frekanslarda çok fazla değiştiği durumlar için kullanımı uygun değil. Bu durumda Redux, MobX gibi store management kütüphaneleri devreye giriyor.

#### useImperativeHandle

`ref` kullanırken, dışarıdan erişilen başka bir component'in özelliğini değiştirmeyi sağlar.

<img height="250" src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhCQbItAcJ7-3A8FgWRIm7VNQd2HHSjsgshbkidNvhLSc6_oV30No2QMQD-UZKaeg7A_65vm84lHk8-YFcfnh1cXg5AJ2E0H2tw3D5WFByE-6JhxhFPwHpIxsdNUXdMqRqR4F3aXt6DA0O7QbFCBmgSeVlXtMKKxBGab0hl6RzI-3EVziSEbbkx9STzBA/s1600/useImperativeHandle.gif">

````jsx
// src\components\Login\Login.js
import React, { useEffect, useReducer, useState, useContext, useRef } from 'react';
import Card from '../UI/Card/Card';
import classes from './Login.module.css';
import Button from '../UI/Button/Button';
import AuthContext from "../../context/auth-context";
import Input from "../UI/Input/Input";

const emailReducer = (state, action) => {
  if (action.type === 'USER_INPUT') {
    return { value: action.val, isValid: action.val.includes('@') };
  }
  return { value: '', isValid: false };
};

const passwordReducer = (state, action) => {
  if (action.type === 'USER_INPUT') {
    return { value: action.val, isValid: action.val.trim().length > 6 };
  }
  return { value: '', isValid: false };
};

const Login = () => {
  const authContext = useContext(AuthContext);

  const [formIsValid, setFormIsValid] = useState(false);

  const [emailState, dispatchEmail] = useReducer(emailReducer, { value: '', isValid: undefined });
  const [passwordState, dispatchPassword] = useReducer(passwordReducer, { value: '', isValid: undefined });

  const emailRef = useRef();
  const passwordRef = useRef();

  useEffect(() => {
    setFormIsValid(
      emailState.isValid && passwordState.isValid
    );
  }, [emailState.isValid, passwordState.isValid]);

  const emailChangeHandler = (event) => {
    dispatchEmail({ type: 'USER_INPUT', val: event.target.value });
  };

  const passwordChangeHandler = (event) => {
    dispatchPassword({ type: 'USER_INPUT', val: event.target.value });
  };

  const submitHandler = (event) => {
    event.preventDefault();

    if (formIsValid) {
      authContext.onLogin(emailState.value, passwordState.value);
    } else if (!emailState.isValid) {
      emailRef.current.focus();
    } else if (!passwordState.isValid) {
      passwordRef.current.focus();
    }
  };

  return (
    <Card className={classes.login}>
      <form onSubmit={submitHandler}>
        <Input
          ref={emailRef}
          type="email"
          id="email"
          label="E-Posta"
          value={emailState.value}
          isValid={emailState.isValid}
          onChange={emailChangeHandler}
        />
        <Input
          ref={passwordRef}
          type="password"
          id="password"
          label="Şifre"
          value={passwordState.value}
          isValid={passwordState.isValid}
          onChange={passwordChangeHandler}
        />
        <div className={classes.actions}>
          <Button type="submit" className={classes.btn}>
            Login
          </Button>
        </div>
      </form>
    </Card>
  );
};

export default Login;


// src\components\UI\Input\Input.js
// forwardRef bir ref'i üst elementten alt elementlerin birine otomatik olarak aktarma tekniğidir.
import { forwardRef, useImperativeHandle, useRef } from "react";
import classes from './Input.module.css';

const Input = forwardRef((props, ref) => {
  const inputRef = useRef();

  const activate = () => {
    inputRef.current.focus();
  };

  useImperativeHandle(ref, () => {
    return {
      focus: activate, // Login'de focus olarak çağırılan metot buradaki activate metodu ile eziliyor.
    };
  });

  return (
    <div
      className={`${classes.control} ${
        props.isValid === false ? classes.invalid : ''
      }`}
    >
      <label htmlFor={props.id}>{props.label}</label>
      <input
        ref={inputRef}
        type={props.type}
        id={props.id}
        value={props.value}
        onChange={props.onChange}
      />
    </div>
  );
});

export default Input;
````





## Appendix

### JavaScript Refresher (ECMA Script)

#### `var`, `let` and `const`

|      | Tanım        | `var`                                              | `let`                                | `const`                                                      |
| ---- | ------------ | -------------------------------------------------- | ------------------------------------ | ------------------------------------------------------------ |
| 1    | Scope        | Function scope özelliğini taşır.                   | Block scope özelliğini taşır.        | Block scope özelliğini taşır.                                |
| 2    | Declaration  | Değişken tekrar tanımlanabilir.                    | Değişken tekrar tanımlanamaz.        | Değişken tekrar tanımlanamaz.                                |
| 3    | Immutability | Değişken değeri güncellenebilir.                   | Değişken değeri güncellenebilir.     | Değişken değeri güncellenemez. Nesne içerisindeki özelliği değiştirilebilir. |
| 4    | Hoisting     | Değişken önce kullanıp, daha sonra tanımlanabilir. | Değişken tanımlanmadan kullanılamaz. | Değişken tanımlanmadan kullanılamaz.                         |

````javascript
function print() {
  // Scope
  if(true)  {
    var varVariable = 'variable 1';
    let letVariable = 'variable 2';
    const constVariable = 'variable 3';
  }

  console.log(varVariable);
  console.log(letVariable); // ReferenceError: letVariable is not defined
  console.log(constVariable); // ReferenceError: constVariable is not defined

    
  // Decleration
  var text1 = 'text1';
  var text1 = 'text01';

  let text2 = 'text2';
  let text2 = 'text02'; // SyntaxError: Identifier 'text2' has already been declared
  
  const text3 = 'text3';
  const text3 = 'text03'; // SyntaxError: Identifier 'text3' has already been declared
    
    
  // Immutability
  var def1 = '1';
  def1 = '1';
  
  let def2 = '2';
  def2 = '2';
  
  const def3 = '3';
  def3 = '3'; // TypeError: Assignment to constant variable.
    
    
  // Hoisting
  number1 = 1;
  console.log(number1);
  var number1;

  number2 = 2;
  console.log(number2);
  let number2; // ReferenceError: Cannot access 'number2' before initialization
    
  number3 = 3;
  console.log(number3);
  const number3; // Missing initializer in const declaration
}

print();
````

#### Arrow Functions

````js
function callMe(name) { 
    console.log(name);
}

const callMe = function(name) { 
    console.log(name);
}

const callMe = (name) => { 
    console.log(name);
}
````

#### Exporting & Importing

<img src="https://blogger.googleusercontent.com/img/a/AVvXsEg-0cacGmraqq8QAtHvhPVgF0cODMolk2y93cxlud3n96Y0B6XmZNrZg3PAe_1K5o9Z8HApunZe04pg76ufU0zVb3zCTt5RcZ59UG16VijMjdGvWb2VZraKwftPOgo2JkKDUbqarPWUQdJ9GMPr_3HRO3rI3Wv-R2UcIIAG98rqHYAfVZv8K-jCBpImGw">

#### Class

````js
class Human {
    gender = 'female';
    
    printGender = () => {
        console.log(this.gender);
    }
} 

class Person extends Human {
    name = 'Hudayfe';
	gender = 'male';
    
    printName = () => {
        console.log(this.nane);
    }
}

const person = new Person();
person.printName();
person.printGender();
````

#### Spread & Rest Operators

````js
// spread for array elements
const numbers = [1, 2, 3];
const newNumbers = [...numbers, 4, 4, 5, [4, 5, 6]];
console.log(newNumbers); // [1, 2, 3, 4, 4, 5, [4, 5, 6]]


// spread for object propeties
const person = {
    name: 'HY'
};
const newPerson = {
    ...person,
    name: 'Hudayfe',
    surname: 'Yurt'
};
console.log(newPerson); 
// {
//   name: "Hudayfe",
//   surname: "Yurt"
// }


// rest for function arguments
const filter = (...args) => {
    return args.filter(el => el === 1);
};
console.log(filter(1, 2, 3, 1, [1, 2])); // [1, 1]
````

#### Destructuring

````js
const numbers = [1, 2, 3];
[num1, , num3] = numbers;
console.log(num1, num3); // 1
						 // 3

const person = {
    name: 'Hudayfe',
    surname: 'Yurt'
};
const {surname} = person;
console.log(surname); // Yurt
````

#### Reference & Primitive Types

````js
const person = {
    name: 'Yurt'
};
const secondPerson = person;

person.name = 'Hudayfe';
console.log(secondPerson); // Hudayfe
````

````js
const person = {
    name: 'Yurt'
};
const secondPerson = {
    ...person
};

person.name = 'Hudayfe';
console.log(secondPerson); // Yurt
````

````js
const numbers = [1, 2, 3];
const doubleNumArray = numbers.map((num) => {
    return num * 2;
});

console.log(numbers); // [1, 2, 3]
console.log(doubleNumArray); // [2, 4, 6]
````



### How the browser renders a web page?

Browser, HTML sayfası için sunucuya istek gönderdiğinde sunucu, HTML sayfasını binary formatta bir text dosyası olarak gönderirken header'da `Content-Type=text-html; charset=UTF-8` bilgisini de döner.

Browser, `text-html` mime type bilgisinden sunucudan gelen cevabın HTML sayfası olduğunu anlar. `charset=UTF-8` bilgisini ise binary formatı, okunabilir formata dönüştürmek için kullanır.

Eğer ki bu header bilgisi sunucudan gelmemiş olursa browser, dosyayı nasıl işleyeceğini ve render'layacağını anlayamaz.



Peki, şu ana kadar herşey istenildiği gibi ilerledi. Sunucudan gelen HTML'de `style.css` ve `main.js` dosyalarıda var. Nasıl oluyorda düz yazı browser'da güzel bir şekilde görünüyor?

````html
<!DOCTYPE HTML>
<html>
    <head>
        <title>Rendering Test</title>
      
        <!-- stylesheet -->
        <link rel="stylesheet" href="./style.css"/>
    </head>
    <body>
        <div class="container">
          <h1>Hello World!</h1>
          <p>This is a sample paragraph.</p>
        </div>
      
        <!-- script -->
        <script src="./main.js"></script>
    </body>
</html>
````

#### Document Object Model (DOM)

Browser, HTML dosyasını okurken HTML element'lerini `Node` adı verilen JavaScript object'ine çevirir.

Her HTML element'i farklı property, class'lara sahiptir. Örneğin `div` element'i `Node` class'ından kalıtım alan `HTMLDivElement` olarak oluşturulur.

Browser, her bir `Node` 'u  bir araya getirerek ağaç yapısını oluşturur.



<img src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiuE9IL_JkMBnWr_r5743MxySlBd3R0zdfWyqGslbvFxmm6FNij-qFSmeJUmIoKRzxJbwX8jbFE5Sz8fFb2Jk6OO8Y_BP7_0gGoNCcMHA2XGTRHMecu9SA7oX6W8NciAVNdr53fNNOCFDX87PmCt5lw2qMI2tW40V5ZzzLN2Kswwvq6k-q4PgbnO9-jpg/s1600/1_YSA8lCfCVPn3d6GWAVokrA.png">

> DOM node'u her zaman HTML element'inden oluşmak zorunda değildir. Yorumlar, attribute'lar ve text'lerden de oluşabilir.

JavaScript, DOM'un ne olduğunu anlayamaz. DOM, web sayfasını oluşturmak ve geliştiricinin DOM öğelerini dinamik olarak değiştirebilmesini sağlayan üst seviye Web API'dir.



> Geliştiriciler, DOM API'yi kullanarak HTML element'i ekleyebilir, kaldırabilir, görünümünü değiştirebilir ve event listener bağlayabilir. DOM API kullanılarak, HTML element'leri bellekte oluşturulabilir, klonlanabilir ve oluşturulan DOM ağacını etkilemeden manipüle edilebilir. Bu, dinamik web sayfası oluşturma yeteneğini sağlar.
>

#### CSS Object Model (CSSOM)

Web sayfasının iyi görünmesi için CSS selector'leri ile DOM element'lerinin stillini değiştiririz.

Browser, DOM'u oluşturduktan sonra, tüm kaynaklardan (external, embedded, inline, user-agent vb.) CSS'i okur ve bir CSSOM oluşturur. CSSOM tıpkı DOM gibi ağaç yapıdadır. Bu ağaçtaki her `Node`, hedeflediği (selector tarafından belirtilen) DOM element'lerine uygulanacak CSS stil bilgilerini içerir.

Browser'ların çoğu user-agent stylesheet olarak adlandırılan kendi style sheet'i ile birlikte gelir. Browser, ilk önce geliştiriciden sağlanan CSS ile user-agent stillerini ezerek DOM element'i için son CSS özelliklerini hesaplar ve ardından bir `Node` oluşturun.

Ne geliştiriciden ne de browser'dan sağlanan bir CSS propert'si yoksa, W3C CSS standardına göre varsayılan property değerleri set'lenir.



Bir CSS property'sinin varsayılan değeri belirlenirken bu property inheritance için uygunsa bazı inheritance kuralları kullanılır. Örneğin, bir HTML element'i için `color` ve `font-size` değerleri eksikse, bu değerleri parent element'inden devralır. Böylece bir HTML element'inin bu özelliklere sahip olduğu ve tüm child element'lerinin onu miras aldığı söylenebilir. Buna stillerin basamaklandırılması denir ve bu nedenle CSS, Cascading Style Sheets'in kısaltmasıdır. Browser'ın, CSS basamaklandırma kurallarına dayalı stilleri hesaplamak için ağaç yapı olan CSSOM'u oluşturmasının nedeni budur.



````css
html {
    padding: 0;
    margin: 0;
}

body {
    font-size: 14px;
}

.container {
    width: 300px;
    height: 200px;
    color: black;
}

.container > h1 {
    color: gray;
}

.container > p {
    font-size: 12px;
    display: none;
}
````

<img src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgL-MlOwaaMwzpDAKHSqtvhQh27b92fZf4MiQLbuHposEmFGhsmpTK_93_oTNlfplt3fw8Yn8xecd_PFxKAQt-2kCr8SV5-KGD21K2l3HNuQvb0v7ku-CfStDT4gTK8GHIQdHYHygnjLm6TKR4NXL6aeNW4ctZBAUuvQ18bJtszLYMjSRBFRy2XtYvU-A/s1600/1_DJg1yRx-AzkZposWbJKcaA.png">

#### Render Tree

Render Tree, browser tarafından DOM ve CSSOM ağaçlarını bir araya getirerek oluşturduğu ağaç yapıdır. Browser, bu Render Tree'yi kullanarak her bir görünür element'in düzenini hesaplamalı ve bunları ekranda boyamalıdır. Bu nedenle, Render Tree oluşturulmadıkça ekrana hiçbir şey yazdırılmayacaktır. Bu yüzden hem DOM hem de CSSOM ağaçlarına ihtiyaç vardır. Render Tree, sonunda ekrana neyin yazdırılacağının düşük seviyeli bir temsili olduğu için, piksel matrisinde herhangi bir alan tutmayan `Node` 'ları içermez. Örneğin, `display:none` element'ler 0px X 0px boyutlarına sahiptir, bu nedenle Render Tree'de bulunmazlar.



<img src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgflPu7qBumHJV6IOXx4FDmHx5UQiNlF_HLGZoLyHzh2aKGUHu5VK0gtnpeXUQHpeNfZbOQBI4x8GG2Hd8muyPTIFW3HQrmiA1OfvCvuGHJMzlwRF65CWmPAGFR_43U9YNtNNavXL3-hUy9gYw7HVgGIZCTW48b9UmdLxVv1mJf0lhXlBWOu2ECKjOcpg/s1600/1_8HnhiojSoPaJAWkruPhDwA.png">



CSSOM, DOM gibi kullanıcının erişebileceği bir yapı değildir. Ancak browser, Render Tree'yi oluşturarak DOM element'inin CSSOM `Node` 'unu ortaya çıkarır. Bu, geliştiricinin CSSOM `Node` 'unun CSS property'lerine erişmesine veya bunları değiştirmesine olanak tanır.

#### Rendering Sequence

Bir web sayfası yüklendiğinde, browser önce HTML metnini okur. Sonrasında DOM ağacı oluşturur. Ardından, ister satır içi ister gömülü veya harici CSS olsun, CSS'i işler ve ondan CSSOM ağacını oluşturur. Bu ağaçları kullanarak Render Tree'yi oluşturur. Render Tree oluşturulduktan sonra browser ekranda tek tek element'leri yazdırmaya başlar.

##### Layout operation

Browser ilk olarak her bir Render Tree `Node` 'unun layout'unu oluşturur. Layout, her bir node'un piksel cinsinden boyutundan ve ekranda nerede (konum) yazdırılacağından oluşur. Browser her node'un layout bilgisini hesapladığı için bu işleme layout veya reflow adı verilir. Scrolling yapıldığında, pencere yeniden boyutlandırıldığında veya DOM element'i değiştirildiğinde de reflow meydana gelir.

Reflow, sayfanın layout'unu hesaplar. Bir element üzerindeki reflow, element'in boyutlarını ve konumunu yeniden hesaplar ve ayrıca o element'in child element'leri, parent'ları ve DOM'da ondan sonra görünen element'ler üzerinde daha fazla reflow'u tetikler. Son olarak son bir repaint çağırır. Reflow çok maliyetlidir ve ne yazık ki kolayca tetiklenebilir. [[Bakınız](https://sites.google.com/site/getsnippet/javascript/dom/repaints-and-reflows-manipulating-the-dom-responsibly)]

Reflow şu durumlarda gerçekleşir:

- DOM'a element eklemek, kaldırmak veya güncellemek
- sayfadaki içeriği değiştirmek, örneğin bir input kutusundaki metin
- bir DOM element'ini taşımak
- bir DOM element'inde animasyon çalıştırmak
- offsetHeight veya getComputedStyle gibi bir element'in ölçümlerini almak
- CSS stilini değiştirmek
- bir element'in className'ini değiştirmek
- stil sayfası eklemek veya kaldırmak
- pencereyi yeniden boyutlandırmak
- scrolling

##### Paint operation

Render Tree'deki element'ler (veya bir sub tree) birbiriyle örtüşebildiğinden ve görünümü, konumu veya geometriyi (animasyonlar gibi) sık sık değiştirmelerini sağlayan CSS özelliklerine sahip olabileceğinden, browser bir layout oluşturur.
Layout oluşturmak, browser'ın browser penceresini kaydırma veya yeniden boyutlandırma gibi bir web sayfasının yaşam döngüsü boyunca boyama işlemlerini verimli bir şekilde gerçekleştirmesine yardımcı olur. Layout'lara sahip olmak, browser'ın element'leri geliştirici tarafından amaçlandığı şekilde yığınlama (stacking order) sırasında (z ekseni boyunca) doğru şekilde çizmesine de yardımcı olur.
Layout'lar oluşturulduğuna göre bunları birleştirilip ekranda çizdirilebilir. Ancak browser tüm layout'ları tek seferde çizmez. Her layout önce ayrı ayrı çizilir.
Browser, her layout'un içinde, element'in `border`, `background-color`, `shadow`, `text` gibi görünür özellikleri ne olursa olsun pikselleri tek tek doldurur. Bu işleme rasterleştirme (rasterization) denir. Browser rasterleştirme performansını artırmak için için farklı thread'ler kullanabilir.

> Rasterleştirmenin CPU'da gerçekleştirilmesi daha yavaş olmasından dolayı günümüzde GPU ile yapılabilmektedir.

##### Compositing operation

Şu ana kadar elimizde farklı layout'lardan (bitmap images) başka bir şey yok. Compositing işleminde bu layout'lar GPU'ya gönderilir ve sonunda ekranda çizdirilir.

Tüm layout'ları çizime göndermek verimsizdir. Bunlar reflow (layout) veya repaint olduğunda oluşturulması gerekir. Bu nedenle, bir layout daha sonra ekranda çizilecek olan farklı tile'lara bölünür. [[Bakınız](https://developer.chrome.com/blog/inside-browser-part3/)]



Critical Rendering Path:

<img src="https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjVngcWIdVNpjXlqByoKn7S1YZR0lAicbC3kVnQ6eC4e4gA3VNnqvP1D8gyvlqxWwyoTsnBRVLF9W_prND21jD5r-Nyrcx990-9t198H_bKwElJYHkEPP8AOqDVYiD0J025-uiaqtrJGu0vEe2MZKb2h3pELf9pDcgAZsg1AgVF6wu0cOa-H_aF5V7Usw/s1600/1_yQJkz12sPxS-kJoMDqzbEQ.png">



##### Browser engines

DOM ağacı, CSSOM ağacı ve rendering mantığı oluşturma işi, browser'da bulunan Browser Engine (Rendering Engine veya Layout Engine olarak da bilinir) adı verilen bir yazılım parçası kullanılarak yapılır. Bu browser motoru, bir web sayfasını HTML kodundan ekrandaki gerçek piksellere dönüştürmek için gerekli tüm element'leri ve mantığı içerir.

#### Rendering Process in Browsers

JavaScript engine sağlayıcıları standartlaştırılmış belirli kurallara uymak zorundadır. Bu standartlara sahip olmak browser'ların tutarlı şekilde JavaScript deneyimi sağlamasını sağlar. Ancak browser'ların rendering işleminde bu durum geçerli değildir.

##### Parsing and External Resources

Parsing işlemi, HTML kodundan DOM ağacı oluşturmadır. HTML kodları sunucudan gelmeye başladığında gelen byte'a göre ekranda render'lama işlemi başlar.

First Paint (FP), browser'ın ekranda bir şeyler yazdırmaya başladığı zaman anlamına gelir (body'nin background renginin ilk pikselinin ekranda boyanması).
First Contentful Paint (FCP), browser'ın metin veya resim gibi içeriğin ilk pikselini oluşturduğu zaman anlamına gelir. Largest Contentful Paint (LCP), browser'ın büyük metin veya resim parçalarını oluşturduğu zaman anlamına gelir.

L, browser tarafından window object'i üzerinde yayılan onload olayını ifade eder. DCL, document object'inde yayılan DOMContentLoaded olayını temsil eder.

DOM parsing işlemi main thread'de gerçekleşir. Ana JavaScript yürütme thread'i meşgulse bu thread serbest kalana kadar, script element'lerinin bloklaması bitene kadar parsing devam etmez.

##### Parser-Blocking Scripts

Parsing engelleme script'i, HTML'nin ayrıştırılmasını durduran bir JavaScript dosyası/kodudur. Browser bir script öğesiyle karşılaştığında, bu gömülü script ise, önce bu script'i çalıştırır ve ardından DOM ağacını oluşturmak için HTML'i ayrıştırmaya devam eder. Yani tüm gömülü script'ler parser'i engeller.

Script, harici bir script dosyasıysa, browser main thread'den harici script dosyasını indirmeye başlar ancak bu dosya indirilene kadar main thread'in yürütülmesini durdurur. JavaScript'in DOM element'lerine erişip bunları değiştirebileceği anlamına gelir. Ancak browser, DOM parsing ve script yürütmeyi paralel olarak çalıştırmak isterse, DOM parser thread'i ile main thread arasında yarış koşulları olabilir. Bu nedenle DOM parsing, main thread ile yapılmalıdır. Ancak, script dosyası arka planda indirilirken DOM parsing'i durdurmak çoğu durumda gereksizdir. Bu nedenle HTML5 bize script etiketi için async attribute'unu verir. DOM parser, async attribute'una sahip harici bir script dosyası element'i ile karşılaştığında, script dosyası arka planda indirilirken parsing işlemini durdurmaz. Ancak dosya indirildikten sonra parsing duracak ve script dosyası (kod) yürütülecektir.
Ayrıca, script element'i için async attribute'una benzer şekilde çalışan ancak async attribute'undan farklı olarak, dosya tamamen indirildiğinde bile script yürütülmeyen defer attribute'una sahibiz.

##### Render-Blocking CSS

Browser, <style> bloğunu bulduğunda tüm gömülü CSS'i ayrıştırır ve CSSOM ağacını yeni CSS kurallarıyla günceller. Sonrasonda, HTML'i normal şekilde ayrıştırmaya devam edecektir. Aynısı satır içi stil için de geçerlidir. Ancak, browser harici bir stylesheet dosyasıyla karşılaştığında parsing'i engellemez, bu nedenle browser arka planda sessizce indirebilir ve DOM parsing devam eder.

Browser'lar harici CSS dosyalarını aşamalı olarak işlemez ve CSSOM ağacı güncellemesi, stil sayfasındaki tüm CSS kuralları işlendikten sonra gerçekleşir. CSSOM ağacı güncellemesi tamamlandığında, Render Tree güncellenir ve ardından ekranda işlenir.

CSS, render engelleyen bir kaynaktır. Browser harici bir stil sayfası getirmek için istekte bulunduğunda, render tree yapımı durdurulur. Bu nedenle, critical rendering path (CRP)'de  takılır ve ekranda hiçbir şey işlenmez. Bununla birlikte, stil sayfası arka planda indirilirken DOM ağacı yapımı devam etmektedir.

Stil sayfası ayrıştırıldıktan ve CSSOM güncellendikten sonra render tree güncellenir ve CRP engeli kaldırılır. Bu da ekranda render tree boyanmasına yol açar. Bu nedenle, tüm harici stil sayfalarının mümkün olduğunca erken, muhtemelen baş bölümünde yüklenmesi önerilir.

Bir stil sayfası arka planda indirilirken, main thread yüklenen stil sayfası tarafından engellenmediği için JavaScript yürütülebilir. JavaScript, bir DOM element'inin CSS property'lerine erişirse (CSSOM API ile), uygun bir değer alırız (default CSSOM durumuna göre). Ancak stil sayfası indirilip ayrıştırıldığında, bu da CSSOM güncellemesine yol açar. Yeni CSSOM güncellemesi o DOM element'inin CSS property'lerini değiştirebileceğinden JavaScript'imiz artık element'in kötü bir CSS property'sine sahiptir. Bu nedenle, stil sayfası indirilirken JavaScript'i yürütmek güvenli değildir.

#### Document’s `DOMContentLoaded` Event

DOMContentLoaded (DCL) olayı, browser'ın mevcut tüm HTML'den eksiksiz bir DOM ağacı oluşturduğu bir zaman noktasını temsil eder.

````javascript
document.addEventListener('DOMContentLoaded', function(e) {
    console.log('DOM is fully parsed!');
});
````

#### Window’s `load` Event

Harici stylesheet ve dosyaların indirildiği ve web uygulamasının indirmeyi bitirdiği bir zaman noktasını temsil eder.

````javascript
window.addEventListener('load', function(e) {
  console.log('Page is fully loaded!');
});
````



### Naming Conventions

| Kural İsmi                                       | Format               |
| ------------------------------------------------ | -------------------- |
| Camel Case (Lower Camel Case, Dromedary Case)    | camelCase            |
| Kebab Case (Dash Case, Lisp Case, Spinal Case)   | kebab-case           |
| Snake Case (Pothole Case)                        | snake_case           |
| Pascal Case (Upper Camel Case, Studly Case)      | PascalCase           |
| Flat Case                                        | flatcase             |
| Upper Flat Case                                  | UPPERFLATCASE        |
| Screaming Snake Case (Macro Case, Constant Case) | SCREAMING_SNAKE_CASE |
| Camel Snake Case                                 | camel_Snake_Case     |
| Pascal Snake Case                                | Pascal_Snake_Case    |
| Train Case (HTTP Header Case)                    | Train-Case           |
| Cobol Case (Screaming Kebab Case)                | COBOL-CASE           |




## Resources

1. [How the browser renders a web page? — DOM, CSSOM, and Rendering](https://medium.com/jspoint/how-the-browser-renders-a-web-page-dom-cssom-and-rendering-df10531c9969#:~:text=When%20a%20web%20page%20is,the%20Render%2DTree%20from%20it)
2. [Repaints and Reflows: Manipulating the DOM responsibly](https://sites.google.com/site/getsnippet/javascript/dom/repaints-and-reflows-manipulating-the-dom-responsibly)
3. [Inside look at modern web browser (part 3)](https://developer.chrome.com/blog/inside-browser-part3/)
4. [How React Works Under the Hood](https://javascript.plainenglish.io/how-react-works-under-the-hood-277356c95e3d)
5. [Değişken İsimlendirme Kuralları](https://juniortoexpert.com/tr/degisken-isimlendirme-kurallari/)
6. [React useRef Hook](https://www.w3schools.com/react/react_useref.asp)
7. [Effect Hook'unu Kullanmak](https://tr.reactjs.org/docs/hooks-effect.html)
8. [Hook'ların API Kaynağı](https://tr.reactjs.org/docs/hooks-reference.html)

