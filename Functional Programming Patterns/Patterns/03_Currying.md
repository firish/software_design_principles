### Currying

Currying is typically referred to as a technique or pattern in functional programming.

Definition: Currying is transforming a function that takes multiple arguments into a function that takes one argument at a time.

Example:
A regular function might look like this in pseudo-code:
```plaintext
function add(x, y) {
  return x + y;
}

Curried version:
function add(x) {
  return function(y) {
    return x + y;
  };
}
Here, add(2) returns a new function that waits for the second argument, so add(2)(3) will return 5.
```

Currying is primarily used in functional programming languages (e.g., Haskell, Scala, and JavaScript when used in a functional style). 
It can also be adopted in other languages that support first-class functions (e.g., Python, Java with lambdas, C#).

Core Idea: It is about decomposing a multi-parameter function into multiple single-parameter functions. 
This approach aligns with the functional paradigm of building small, composable units.

### Why Is It Used?

1. Partial Application:
- Currying makes partial application straightforward.
- For instance, if you have a function multiply(a, b, c), you could fix a and b in advance, returning a new function waiting for the final argument.
- This allows reusing or specializing functions without rewriting or duplicating them.

2. Function Composition:
- In functional programming, composing functions is key.
- Curried functions integrate neatly with other functional features like map, filter, reduce, and function pipelines.
- When used judiciously, it can lead to more declarative and modular code, where each curried step is logically separated.
  
3. Immutability and Referential Transparency:
- Each curried call is essentially a new function without side effects, promoting immutability and making code easier to reason about in a functional style.

4. Easier Testing
- Because you can partially apply functions and test them at each step, it can simplify writing unit tests for smaller pieces of logic.
- Each call explicitly provides one argument, making it very clear how arguments are transformed or passed along.


### Example Usage

Imagine you have an e-commerce scenario where you need to calculate the final price of a product. 
This calculation depends on multiple factors such as the original price, a discount percentage, and a sales tax rate.

We’ll first look at non-curried code, then see how we might rewrite it in a curried style.
```js
/**
 * Calculates the final price of a product.
 * 
 * @param {number} originalPrice - The base price of the product.
 * @param {number} discountRate  - The discount rate (e.g., 0.1 for 10% discount).
 * @param {number} taxRate       - The tax rate (e.g., 0.07 for 7% tax).
 * @returns {number} The final price after discount and tax.
 */
function calculateFinalPrice(originalPrice, discountRate, taxRate) {
  // Apply discount
  const priceAfterDiscount = originalPrice * (1 - discountRate);

  // Apply tax
  const finalPrice = priceAfterDiscount * (1 + taxRate);

  return finalPrice;
}

// Example usage:
const productPrice = 100;    // $100
const discount10Percent = 0.1; 
const tax7Percent = 0.07;

const finalPrice = calculateFinalPrice(productPrice, discount10Percent, tax7Percent);
console.log(finalPrice); // Output: 96.3
```

What Happens Here?
We pass all three arguments (originalPrice, discountRate, taxRate) every time we call calculateFinalPrice.
If you want to reuse the same discount rate or tax rate across multiple calculations, you must repeatedly pass them in, or you must create your own wrapper function.


2. With Currying
In the curried approach, you transform your function so it takes one argument at a time and returns a new function expecting the remaining arguments. This approach makes partial application easier—where you “fix” certain arguments and reuse the resulting function with fewer parameters.

```js
/**
 * Curried function to calculate the final price in steps.
 * 
 * @param {number} originalPrice - The base price of the product.
 * @returns {function} A function waiting for the discount rate.
 */
function curriedCalculateFinalPrice(originalPrice) {
  // Return a function expecting 'discountRate'
  return function(discountRate) {
    // Return another function expecting 'taxRate'
    return function(taxRate) {
      const priceAfterDiscount = originalPrice * (1 - discountRate);
      const finalPrice = priceAfterDiscount * (1 + taxRate);
      return finalPrice;
    };
  };
}

// Example usage:
const productPriceCurried = 100;
const discount10PercentCurried = 0.1; 
const tax7PercentCurried = 0.07;

// Step 1: Partially apply the original price.
const withOriginalPrice = curriedCalculateFinalPrice(productPriceCurried);

// Step 2: Partially apply the discount rate.
const withOriginalPriceAndDiscount = withOriginalPrice(discount10PercentCurried);

// Step 3: Finally apply the tax rate to get the final price.
const finalPriceCurried = withOriginalPriceAndDiscount(tax7PercentCurried);

console.log(finalPriceCurried); // Output: 96.3

// OR
const finalPriceSingleLine = curriedCalculateFinalPrice(100)(0.1)(0.07);
console.log(finalPriceSingleLine); // Output: 96.3
```

What’s Happening in the Curried Code?
First Function Call: curriedCalculateFinalPrice(productPriceCurried)
Returns a new function waiting for the discount rate.
Second Function Call: (discountRateValue)
Returns yet another function waiting for the tax rate.
Third Function Call: (taxRateValue)
Returns the final price computed using all three values.


Why Use the Curried Version?
Partial Application: You can create specialized versions of the function by partially applying some arguments. 

For example, below is one way to set up a curried function so that you can predefine your percentages (discount, tax, payment gateway fee) and then call the resulting function with only the original price. This example uses partial application to “lock in” your fixed rates.

1. Define the Curried Function
We need a function that takes these four parameters in a curried manner:

Discount Rate (fixed at 20%)
Tax Rate (fixed at 8%)
Payment Gateway Fee (fixed at 2%)
Original Price (variable)
```js

function curriedCalculateFinalPrice(discountRate) {
  return function(taxRate) {
    return function(paymentGatewayFee) {
      // Now we return a function waiting for the originalPrice
      return function(originalPrice) {
        // Step 1: Apply discount
        const priceAfterDiscount = originalPrice * (1 - discountRate);
        
        // Step 2: Apply tax
        const priceAfterTax = priceAfterDiscount * (1 + taxRate);

        // Step 3: Apply payment gateway fee
        const finalPrice = priceAfterTax * (1 + paymentGatewayFee);

        // Return the final price
        return finalPrice;
      };
    };
  };
}
```

Why Structure It This Way?
By ordering the parameters in a chain, we can partially apply the first three (which are fixed) and end up with a function that only needs the originalPrice. This technique is often used in functional programming to create specialized functions by pre-supplying some arguments.

2. Create a Specialized Function with Fixed Rates
Since you want 20% discount (0.20), 8% tax (0.08), and 2% payment gateway fee (0.02), you can do:

```js
// 20% discount -> 0.20
// 8% tax       -> 0.08
// 2% fee       -> 0.02

const finalPriceCalc = curriedCalculateFinalPrice(0.20)(0.08)(0.02);

const originalPrices = [50, 80, 100, 120, 150, 200, 220, 300, 350, 400];

// Map over each original price, calling the preconfigured function
const finalPrices = originalPrices.map((price) => finalPriceCalc(price));

console.log(finalPrices);
/*
  Example output (approximate):
  [
    51.84,   // final price for original price of 50
    82.94,   // final price for 80
    103.68,  // final price for 100
    124.42,  // final price for 120
    155.52,  // final price for 150
    207.36,  // final price for 200
    228.09,  // final price for 220
    311.04,  // final price for 300
    362.88,  // final price for 350
    414.72   // final price for 400
  ]
*/
```
