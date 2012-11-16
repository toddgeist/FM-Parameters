# FM-Parameters

FM-Parameters is a series of tools, mostly custom functions, for creating and parsing serialized data objects in FileMaker. These are most frequently used for passing multiple pieces of data in script parameters and results. This repository is an incomplete work in progress, based on the recommended script parameter interface of [FileMakerStandards.org][1].

[1]: http://filemakerstandards.org/pages/viewpage.action?pageId=557462 "FileMakerStandards.org: Script Parameter Interface"

## The format

These functions manipulate data stored in a format that matches the variable-declaration format of FileMaker's Let () function. This is so that the format will appear familiar to FileMaker developers without additional background knowledge. This mirrors the relationship between JavaScript and JSON.

## The functions

### # ( name ; value )

The # ( name ; value ) function creates a Let notation name-value pair. A dictionary data structure can be created by concatenating several calls to #() as if they were plain text. Let notation also supports nested dictionaries.

	# ( "name" ; "value" )
	& # ( "outerName" ;
		# ( "innerName" ; "inner value" )
	)

Named values can be over-written or effectively erased by concatenating a call to #() to the end of a dictionary using the same name and a different value. This works because the #Get and #Assign functions will always respect the *last* instance of a name-value pair in a dictionary.

	# ( "name" ; "value" )
	& # ( "foo" ; "bar" )
	& # ( "name" ; "new value" )
	& # ( "foo" ; "" )	// Assigning null value effectively deletes "foo"

This last-value-wins behavior can also be used to set default values for optional parameters.

	# ( "optionalParameter" ; "default value" )
	& Get ( ScriptParameter )

By placing the defaults before the actual parameters, any values set by the actual script parameter will override the defaults.

### #Assign ( parameters )

The #Assign functions parse a Let format dictionary into local script variables. The name from each name-value pair is used as the variable name, and the value from each pair is set to that variable's value.

	#Assign ( # ( "name" ; "value" ) )	// variable $name assigned "value"

The #Assign function returns True (1) if there was no error detected while assigning the values to variables, and returns False (0) otherwise. If there was an error detected, FileMaker's error code is assigned to the $#Assign.error variable.

Some legacy versions of the # ( name ; value ) function allow developers to set names intended for global variables. The # ( name ; value ) function defined here will not allow that, but the #Assign function currently will still set legacy global variables in dictionaries for backwards compatibility. This is likely to change in the future.

### #Filter ( parameters ; filterParameters )

The #Filter function returns a Let format dictionary containing only those name-value pairs where the name is included in the return-delimited list filterParameters.

	#Assign ( #Filter (
		# ( "name" ; "value" )
		& # ( "foo" ; "bar" );
		List ( "name" ; "otherName" )
	) )	// variable $name assigned "value"; $foo and $otherName are unaffected

This function can prevent an "injection" of unexpected variables that might cause problems.

### #AssignGlobal ( parameters )

The #AssignGlobal function parses a Let format dictionary into global script variables

	#AssignGlobal ( # ( "name" ; "value" ) )	// variable $$name assigned "value"

This approach to setting global variables is preferred over other methods that allow the dictionary itself to define what values should be assigned to globals because it makes more sense that the code that actually assigns the variables should have discretion over what type of variable gets assigned.

The #AssignGlobal function return True (1) if there was no error detected while assigning the values to variables, and returns False (0) otherwise. If there was an error detected, FileMaker's error code is assigned to the $#AssignGlobal.error variable.

### #Get ( parameters ; name )

The #Get function returns a named value from a dictionary.

	#Get ( # ( "name" ; "value" ) ; "name" )	// = "value"

Unlike the #Assign functions, #Get will not affect any variables, which can be useful when a named value only needs to be used in one calculation, or to assign a named value to a variable with a different name.

### VerifyVariablesNotEmpty ( parameterNameList )

Returns True (1) if each of the parameters in parameterNameList has been assigned to a non-empty local script variable of the same name. Returns False (0) if any variable defined by parameterNameList is empty.

	VariablesNotEmpty ( List ( "parameter1" ; "parameter2" ) )

This is useful for verifying that each parameter required by a script has been successfully assigned before proceeding.

### ScriptRequiredParameterList ( scriptNameToParse )

Parses a script name, returning a return-delimited list of parameters required for that script. This function assumes that the script name conforms to the FileMakerStandards.org [naming convention for scripts][2]. This is useful to generate the argument used by the VariablesNotEmpty function to validate that all required parameters have values. In practice, developers may want to modify this function to always use the current Get ( ScriptName ) and not require the argument.

[2]: http://filemakerstandards.org/display/cs/Script+naming "FileMakerStandards.org: Script naming"

### ScriptOptionalParameterList ( scriptNameToParse )

Parses a script name, returning a return-delimited list of optional parameters for that script. This function assumes that the script name conforms to the FileMakerStandards.org [naming convention for scripts][2]. This is useful to generate the argument used by the #AssignWhiteList function to restrict variable assignment to parameters actually accepted by a script. In practice, developers may want to modify this function to always use the current Get ( ScriptName ) and not require the argument.

	#AssignWhiteList (
		Get ( ScriptParameter );
		ScriptRequiredParameterList ( Get ( ScriptName ) )
		& ScriptOptionalParameterList ( Get ( ScriptName ) )
	)

## Deprecated functions

These functions are implemented for the sake of backwards compatibility in those solutions using the current FileMakerStandards.org functions, but their use is discouraged. These functions are all equivalent to simple combinations of other functions. These functions may be easier to type than the equivalent combinations of other functions, but tools like [TextExpander][], [Breevy][], and [Clip Manager][] make this a moot point. This approach also makes the resulting code more explicit about what logic is being applied to what data inputs, making the code more readable, especially to developers unfamiliar with the functions.

[TextExpander]: http://smilesoftware.com/TextExpander/index.html
[Breevy]: http://www.16software.com/breevy/
[Clip Manager]: http://www.myfmbutler.com/index.lasso?p=422

### #AssignScriptParameters

The #AssignScriptParameters function will assign all named values in the script parameter to local script variables of the same name. If any parameters indicated as required by the script name are empty, the function returns False (0); the function returns True (1) otherwise.

	If [not #AssignScriptParameters]
		Exit Script [Result:# ( "error" ; 10 )	// Requested data is missing]
	End If

A combination of the #Assign, VariablesNotEmpty, and ScriptRequiredParameterList functions is the preferred way to replicate this behavior:

	Set Variable [$ignoreMe; Value:#Assign ( Get ( ScriptParameter ) )]
	If [not VerifyVariablesNotEmpty ( ScriptRequiredParameterList ( Get ( ScriptName ) ) )]
		Exit Script [Result:# ( "error" ; 10 )	// Requested data is missing]
	End If

This approach enables greater flexibility in defining what variables are required and how those variables are assigned.

### #AssignScriptResults

This function is exactly equivalent to this calculation:

	#Assign ( Get ( ScriptResult ) )

Using this calculation instead of this function is preferred.

### #GetScriptParameter ( name )

This function is exactly equivalent to this calculation:

	#Get ( Get ( ScriptParameter ) ; name )

Using this calculation instead of this function is preferred.

### #GetScriptResult

This function is exactly equivalent to this calculation:

	#Get ( Get ( ScriptResult ) ; name )

Using this calculation instead of this function is preferred.

## Acknowledgements

The Let format was inspired by an example in FileMaker's [documentation for the Let() function][3].

[3]: http://www.filemaker.com/help/html/func_ref3.33.15.html "FileMaker, Inc.: Let"

The dictionary functions by Six Fried Rice (mentioned in [introductory][4] and [supplemental][5] posts) were a significant inspiration for the behaviors of some functions and the "#" prefix notation.

[4]: http://sixfriedrice.com/wp/passing-multiple-parameters-to-scripts-advanced/ "Six Fried Rice: Passing Multiple Parameters to Scripts - Advanced"
[5]: http://sixfriedrice.com/wp/filemaker-dictionary-functions/ "Six Fried Rice: FileMaker Dictionary Functions"

Dan Smith started the correspondence that inspired me to reform the existing FileMakerStandards.org solution for object serialization.

## License

Anyone may do anything with this software. There is no warranty.