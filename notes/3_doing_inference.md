# Doing inference 

- Test the different code snippets by navigating here: [webppl.org](http://webppl.org)
- You can find a complete function reference here: [http://webppl.readthedocs.io/en/master/](http://webppl.readthedocs.io/en/master/)
- Check out this chapter of the probmods book for more information: [03-conditioning](https://probmods.org/chapters/03-conditioning.html)

## Conditioning on variables 

### Rejection sampling 

- We can use recursion to implement rejection query.
- Here we are interested in the value of `a`, conditioning on the fact that the sum of `a`, `b`, and `c` is `<= 2`.
- Note how the `flippingAway()` function is called inside the `flippingAway()` function. The return statement is an if-else statement. If the condition is met `d <= 2`, the value of `a` is returned, otherwise the `flippingAway()` function is called again. 

```javascript
var flippingAway = function () {
	var a = flip(0.3)
	var b = flip(0.3)
	var c = flip(0.3)
	var d = a + b + c
	return d <= 2 ? a : flippingAway()
}
viz(repeat(100, flippingAway))
```

### Using WebPPL's inference procedures 

- To do inference, we simply add a `condition` statement to our model. 
- The general template is: 

```javascript
var model = function(){
	var ... //our model description
	condition (...) // what we condition on
	return ... // what we are interested in
}
```

- To run inference, we use the webppl procedure `Infer`. 
- `Infer` takes an inference procedure, and a function as input. 
- Let's see what happens if we condition on the fact that `d <= 2`.

```javascript
var flippingAway = function(){
	var a = flip(0.3)
	var b = flip(0.3)
	var c = flip(0.3)
	var d = a + b + c
	condition(d <= 2) //condition
	return d
}
var options = {method: 'rejection'}
var dist = Infer(options, flippingAway)
viz(dist)
```

- You can compare this posterior distribution to the prior distribution by simply changing the `condition(d <= 2)` to `condition(true)` (or by commenting out the `condition(...)` statement). 

## Conditioning on arbitrary expressions 

- One of the powerful features of the webppl programming language is that it allows conditioning on arbitrary expressions composed of the variables in the model. 
- You can also return arbitrary expressions. 
- Here is an example: 

```javascript
var flippingAway = function(){
	var a = flip(0.3)
	var b = flip(0.3)
	var c = flip(0.3)
	condition(a + b + c <= 2) //arbitray expression
	return a + b + c //arbitrary expression
}
var options = {method: 'rejection'}
var dist = Infer(options, flippingAway)
viz(dist)
```

- This is one key strength since it allows us to cleanly separate out the description of the generative model, and the inference procedure. 

## Other inference procedures 

- In the previous examples, we used enumeration (`model: `enumerate``) to do inference. This was only feasible since the space of possible program executions was rather small. 
- WebPPL implements a number of inference procedures. You can find more abou these here: [http://webppl.readthedocs.io/en/master/inference/methods.html](http://webppl.readthedocs.io/en/master/inference/methods.html)
- TODO: mention that MH is one of the most used procedures ... 

## Practice 

### Inference in the tug of war model

- You now have learned about all the bits and pieces that make up the tug of war model. 
- Here is the model: 

```javascript
var model = function(){
	var strength = mem(function (person) {return gaussian(50, 10)})
	var lazy = function(person) {return flip(1/3) }
	var pulling = function(person) {
			return lazy(person) ? strength(person) / 2 : strength(person) 
			}
	var totalPulling = function (team) {return sum(map(pulling, team))}
	var winner = function (team1, team2) {
			totalPulling(team1) > totalPulling(team2) ? team1 : team2
			}
	var beat = function(team1,team2){winner(team1,team2) == team1}
	// Condition and Return statements go here ...
}
// Infer and visualize here ...
```

- First, make sure that you understand all the bits and pieces. 
- Condition on the fact that `"Tom"` beat `"Bill"`, and return the strength of `"Tom"`. (Note: The `beat` function takes teams as input (i.e. arrays). So even if the team only has one player, you still need to put that player into an array.)
- For the inference options, please use the following: `var options = {{method: 'MCMC', kernel: 'MH', samples: 25000}`. This implements a Markov Chain Monte Carlo inference. 
- If all goes well, the `viz` function will output a density function. You can print out the mean of the distribution by using the `expectation()` function: `print('Expected strength: ' + expectation(dist))`

<!--
- SOLUTION:

 ```javascript
var model = function() {
	//MODEL
	var strength = mem(function (person) {return gaussian(50, 10)})
	var lazy = function(person) {return flip(1/3) }
	var pulling = function(person) {
		return lazy(person) ? strength(person) / 2 : strength(person) }
	var totalPulling = function (team) {return sum(map(pulling, team))}
	var winner = function (team1, team2) {
		totalPulling(team1) > totalPulling(team2) ? team1 : team2 }
	var beat = function(team1,team2){winner(team1,team2) == team1}
	
	//CONDITION	
	condition(beat(['Tom'], ['Steve','Bill']))
	
	//QUERY
	return strength('Tom')
}
var options = {method: 'MCMC', kernel: 'MH', samples: 25000}
var dist = Infer(options,
								 model)

viz(dist)
print('Expected strength: ' + expectation(dist))
``` -->

### Extending the tug of war model 

- Extend the tug of war model so that you can ask whether a player was lazy in a particular match. 
- To do so, you need to amend the functions to not only feature persons (or teams) but also matches. 
- You need to make laziness a persistent property (using `mem`) that applies to a person in a match. 
- Once you've rewritten the code then try the following: 
	+ Condition on the fact that Tom beat Tim in match 1 (hint: `condition(beat(['Tom'],['Tim'],1))`), and ask for whether Tom was lazy in match 1 (and whether Tim was lazy in match 1). 
	+ How does the inference whether Tim was lazy in match1 change for the following series of matches?: match 1: Tim loses against Tom; match 2: Tim wins against Steve; match 3: Tim wins against Bill; match 4: Tim wins against Mark. (Note: Use `&` to combine multiple pieces of evidence in the condition statement `condition()`).

<!-- 
- SOLUTION: 

```javascript

var model = function(){
	//MODEL 
	var strength = mem(function(person) {
		return gaussian(50, 10)
	})

	var lazy = mem(function(person, match) {
		return flip(0.3)
	})

	var pulling = function(person, match) {
		return lazy(person, match) ? strength(person) / 2 : strength(person) 
	}

	var totalPulling = function(team, match) {
		return sum(map(function(person) {
			return pulling(person, match)
		}, team))
	}

	var winner = function(team1, team2, match) {
		return totalPulling(team1, match) > totalPulling(team2, match) ? team1 : team2
	}

	var beat = function(team1,team2, match) {
		return winner(team1,team2, match) == team1
	}
	
	// CONDITION 
	condition(
		beat(['Tom'],['Tim'],1) & 
		beat(['Tim'],['Steve'],2) &
		beat(['Tim'],['Bill'],3) &
		beat(['Tim'],['Mark'],4)
	)

	//QUERY 
	return lazy('Tim',1)
}
var options = {method: 'MCMC', kernel: 'MH', samples: 25000}
var dist = Infer(options,model)

viz(dist)
```
 -->