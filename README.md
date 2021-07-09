# JSValid

JSValid is a functional approach to validation in JavaScript. It provides a natural means of expressing value constraints, as well as a simple mechanism for enforcing them.

JSValid specifies the signature of a special kind of function, called a __validator__. Validators accept as a single argument the __subject__, which is the value to be validated, and return a report.

A __report__ is an object with a `violations` property, which is an array of violations. An empty violations array indicates a valid subject. A validator may augment its report with additional properties.

A __violation__ is an object with the following properties, of which only `message` is mandatory:

- `message`: A string describing the violation.
- `path`: An array of keys describing where the violation occurred (for example, `["people", 0]`).
- `code`: A string uniquely identifying the type of violation.
- `a`: Exhibit A.
- `b`: Exhibit B.

A __factory__ is any function which returns a validator function. Here is a demonstration showing how factories, validators and reports work together.

    // A validator is made by invoking a factory.

    const my_validator = valid.object({
        question: valid.string(),
        answer: valid.integer()
    });

    // A report is generated by invoking the validator with a subject.

    const my_report = my_validator({
        question: "What is the answer to life, the universe and everything?",
        answer: "Love."
    });

    // The report may be inspected for violations.

    // {
    //     violations: [
    //         {
    //             message: "Not of type integer.",
    //             path: ["answer"]
    //             code: "not_type_a",
    //             a: "integer"
    //         }
    //     ]
    // }

## The factories

An object containing several factory functions is exported by jsvalid.js:

    import valid from "./jsvalid.js";
    const {
        boolean,
        number,
        integer,
        string,
        function,
        array,
        object,

        wun_of,
        all_of,
        literal,
        any
    } = valid;

Incidentally, these factories complement the specifiers provided by [JSCheck](https://www.crockford.com/jscheck.html), a testing tool written by Douglas Crockford.

### valid.boolean()

The __boolean__ validator permits only `true` and `false`.

### valid.number()

The __number__ validator permits only numbers which satisfy `Number.isFinite`. This excludes `Infinity` and `NaN`.

### valid.number(_minimum_, _maximum_)

Specifying either of _minimum_ or _maximum_ imposes inclusive bounds on the subject.

    function valid_longitude() {
        return valid.number(-180, 180);
    }
    function valid_latitude() {
        return valid.number(-90, 90);
    }
    function valid_weight() {
        return valid.number(0);
    }

### valid.integer()

The __integer__ validator permits only numbers which satisfy `Number.isSafeInteger`.

### valid.integer(_minimum_, _maximum_)

See `valid.number`.

### valid.string()

The __string__ validator permits only strings.

### valid.string(_regular_expression_)

The subject must conform to the _regular_expression_.

    function valid_tracking_number() {
        return valid.string(/^[0-9]{24}$/);
    }

### valid.string(_length_validator_)

The length of the subject must conform to the _length_validator_.

    function valid_note() {
        return valid.string(valid.integer(1, 140));
    }

### valid.function(_length_validator_)

The __function__ validator permits only functions. The arity of the subject must conform to the _length_validator_, if it is specified.

### valid.array()

The __array__ validator permits only arrays, as determined by `Array.isArray`.

### valid.array(_validator_, _length_validator_)

Each element in the subject must conform to the _validator_. The length of the array must conform to the _length_validator_, if it is specified.

### valid.array(_validator_array_, _length_validator_, _rest_validator_)

Each element in the subject must conform to the validator at the corresponding position in the _validator_array_. If the _length_validator_ is omitted, the subject must be the same length as the _validator_array_.

    function valid_location() {
        return valid.array([valid_longitude(), valid_latitude()]);
    }
    function valid_mail_journey() {
        return valid.array(valid_location(), valid.integer(1));
    }

It is possible that the _length_validator_ may permit a subject longer than the _validator_array_. In such a case, each surplus element of the subject must conform to the _rest_validator_. If the _rest_validator_ is undefined, the sequence of surplus elements must conform to the sequence of validators formed by repeating the _validator_array_.

### valid.object()

The __object__ validator permits only bona fide objects, not `null` or arrays.

### valid.object(_required_properties_, _optional_properties_, _allow_strays_)

The _required_properties_ and _optional_properties_ parameters are objects containing validators. Either parameter may be undefined.

The value of each property on the subject must conform to the corresponding validator in _required_properties_ or _optional_properties_. Where no corresponding validator is found, the property is permitted only if _allow_strays_ is `true`. Additionally, the subject must contain every key found on _required_properties_.

`Object.keys` is used to enumerate the subject's properties, as well as the validators in _required_properties_ and _optional_properties_.

    function valid_parcel() {
        return valid.object(
            {
                id: valid_tracking_number(),
                size: "parcel",
                weight: valid_weight()
            },
            {
                delivery_advice: valid_note()
            }
        );
    }

### valid.object(_key_validator_, _value_validator_)

Each key found on the subject by `Object.keys` must conform to the _key_validator_. Each corresponding value must conform to the _value_validator_.

All keys are permitted if the _key_validator_ is undefined. Likewise, all values are permitted if the _value_validator_ is undefined.

    function valid_tracking_info() {
        return valid.object(valid_tracking_number(), valid_mail_journey());
    }

### valid.wun_of(_validator_array_)

The __wun_of__ validator permits only values which conform to at least wun of the validators in the _validator_array_.

    function valid_size() {
        return valid.wun_of(["letter", "parcel", "postcard"]);
    }

A definition of the word "wun" may be found [here](http://howjavascriptworks.com/sample.html).

### valid.all_of(_validator_array_, _exhaustive_)

The __all_of__ validator permits only values which conform to every validator in the _validator_array_. It runs the validators in sequence, stopping at the first violation (unless _exhaustive_ is `true`).

### valid.literal(_expected_value_)

The __literal__ validator permits only values equal to the _expected_value_. The `===` operator is used to determine equality unless either of the values is `NaN`, in which case `Number.isNaN` is used.

Some factories provided by JSValid accept validators as arguments. Where a non-function is provided in place of a validator, it is automatically wrapped with `valid.literal`. Thus, the expression

    valid.string(valid.literal(1));

may be written more succinctly as

    valid.string(1);

Consequently, `valid.literal` is only needed in cases where _expected_value_ is a function or `undefined`.

### valid.any()

The __any__ validator permits any value.

## But what about...?

It is easy to make your own validators. Here is a factory which returns a validator which permits only multiples of _n_.

    function valid_multiple_of(n) {
        return function multiple_of_validator(subject) {
            return {
                violations: (
                    subject % n === 0
                    ? []
                    : [{message: `Not a multiple of ${n}.`}]
                )
            };
        };
    }
