+++
title = 'The Frustration of Repetition'
date = 2024-06-09T09:34:23+05:30
draft = false
summary = "There comes a time when you get so bored doing some daily boring and repeative tasks during coding, eventually get fedup enough that you spend the next weekend to do something about it."
+++

## The Frustration of Repetition : Automating JSON Randomization

We've all been there: staring down a mountain of repetitive tasks that stifle our creativity and slow down development.
During development or testing, we often need to randomize JSON input based on a specific template. For example, when testing a POST endpoint locally, you might need to repeatedly update a value in your Postman or event file to meet unique constraints or de-duplication logic. This process can be tedious and time-consuming. I found myself in this exact situation over the past few months while working intensively on a serverless project.
Each request needed a unique ID, and every code change required updating the `events.json` file used by the AWS SAM CLI. It was incredibly frustrating! After failing to find a tool that could handle template-based JSON randomization, I decided to take matters into my own hands. Here's how I did it.

### Planning it out

I wanted a tool that would let me define a JSON template and update only a few keys with specific types of values and lengths. Additionally, it’s crucial that the data falls within user-defined constraints—such as a string being exactly 25 characters long or an integer being between 3000 and 4000. Also, the data can be nested which also needs to be adressed.

For instance, consider the JSON below. Every time, I need a unique request ID (a UUID), an age between 20 and 30, and an OTP that is a 4-digit integer.

By creating a template file with placeholders, I can automate this process:

```json
{
  "name": "rohan",
  "request_id": "$UUID",
  "age": "$INT(20:30)",
  "product_price": "$INT",
  "rand_char": "$CHAR(100)",
  "data": {
    "token": "$CHAR(100)",
    "OTP": "$INT(1000:9999)"
  }
}
```

This template makes it easy to generate randomized data that meets specific constraints, streamlining development and testing while reducing repetitive tasks.

### Code walkthrough !

Github Repo :  
[https://github.com/rohanchavan1918/gorandomify](https://github.com/rohanchavan1918/gorandomify)

This Go code defines a command-line tool that accepts input and output file paths as flags. It first checks if a source template file is provided; if not, it looks for input data passed through system arguments. It reads the input data, unmarshals it into a map, and then copies the data for manipulation. After traversing and updating the copied data, it marshals it back into JSON format. If an output file path is specified, it writes the updated JSON to that file; otherwise, it prints it to the console. This tool efficiently processes JSON data, offering flexibility in input methods and output destinations.

```go
func main() {
	sourcePath := flag.String("t", "", "Source of template file")
	destinationPath := flag.String("o", "", "Destination path")
	flag.Parse()

	var passedFromSysArgs bool = false
	var inputData []byte

	if *sourcePath == "" {

		if len(os.Args) < 2 {
			colorize(ColorRed, "no input or template file passed")
			os.Exit(1)
		}
		passedFromSysArgs = true

	}

	if passedFromSysArgs {
		inputData = []byte(os.Args[1])
	} else {
		var err error
		inputData, err = os.ReadFile(*sourcePath)
		if err != nil {
			colorize(ColorRed, err.Error())
			return
		}
	}

	var originalData map[string]interface{}
	if err := json.Unmarshal(inputData, &originalData); err != nil {
		colorize(ColorRed, err.Error())
		return
	}

	copiedData := copyData(originalData)
	traverseAndUpdate(originalData, copiedData)

	updatedJSON, err := json.MarshalIndent(copiedData, "", "  ")
	if err != nil {
		colorize(ColorRed, "Error: "+err.Error())
		return
	}

	if *destinationPath != "" {
		if err := writeToFile(*destinationPath, updatedJSON); err != nil {
			colorize(ColorRed, "Error: "+err.Error())
			return
		}

		colorize(ColorGreen, "JSON generated successfully: "+*destinationPath)
	} else {
		fmt.Println(string(updatedJSON))
	}

}
```

The `traverseAndUpdate` function recursively traverses through the JSON data, updating values as needed. For each key-value pair, it checks the type of the value. If the value is a nested map, it recursively calls `traverseAndUpdate` to handle it. If the value is a string, it invokes the `parseAndUpdate` function to update it based on certain conditions.

The `parseAndUpdate` function takes a key-value pair, along with the original and copied data maps. It identifies if an updater function is available for the value and, if so, attempts to update the value accordingly. If successful, it updates both the original and copied data maps. If an error occurs during updating, it prints a red-colored error message. These helper functions facilitate the main functionality of the tool by handling data traversal, file writing, and value updating.

```go
func traverseAndUpdate(data, copiedData map[string]interface{}) {
	for key, value := range data {
		switch v := value.(type) {
		case map[string]interface{}:
			traverseAndUpdate(v, copiedData[key].(map[string]interface{}))
		case string:
			parseAndUpdate(key, v, data, copiedData)
		}
	}
}

func writeToFile(filename string, data []byte) error {
	return ioutil.WriteFile(filename, data, 0644)
}

func parseAndUpdate(key, value string, data, copiedData map[string]interface{}) {
	if updater := getUpdater(value); updater != nil {
		if newVal, err := updater.Update(value); err == nil {
			data[key] = newVal
			copiedData[key] = newVal
		} else {
			colorize(ColorRed, "Error updating key: "+key+", value: "+value+", err: "+err.Error())
		}
	}
}
```

The `getUpdater` function determines the appropriate updater for a given value string. It checks the prefix of the value string to identify the type of update required. If the value starts with "$UUID", it returns the UUID updater; if it starts with "$INT", it returns the integer updater; and if it starts with "$CHAR", it returns the character updater. If none of these conditions match, indicating an unsupported type, it returns nil.

The `Updater` interface defines a method `Update` that takes a string value and returns an interface and an error. This interface is implemented by three updater structs: `UUIDUpdater`, `IntUpdater`, and `CharUpdater`, each responsible for generating a UUID, parsing an integer, or generating a random string, respectively.

These updaters are stored in a map named `updaters`, with keys corresponding to the supported types and values being instances of the updater structs. When `getUpdater` identifies the type of updater needed, it retrieves the corresponding updater from this map and returns it.

Overall, `getUpdater` plays a crucial role in determining the appropriate updater method based on the value string, facilitating the dynamic updating of JSON data based on predefined rules and types.

```go
func getInt(value string) (int, error) {
	matches := regexp.MustCompile(`^\$INT\((\d+):(\d+)\)$`).FindStringSubmatch(value)
	if len(matches) == 0 {
		return rand.Intn(10000), nil
	}

	lower, err1 := strconv.Atoi(matches[1])
	upper, err2 := strconv.Atoi(matches[2])
	if err1 != nil || err2 != nil || lower > upper {
		return 0, fmt.Errorf("invalid INT range")
	}

	return rand.Intn(upper-lower+1) + lower, nil
}

func randomString(value string) (string, error) {
	length := 10
	if parts := strings.Split(value, "("); len(parts) == 2 {
		if l, err := strconv.Atoi(strings.TrimSuffix(parts[1], ")")); err == nil {
			length = l
		}
	}
	return getRandomStrNlen(length), nil
}

func getRandomStrNlen(n int) string {
	const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
	rand.Seed(time.Now().UnixNano())

	result := make([]byte, n)
	for i := range result {
		result[i] = charset[rand.Intn(len(charset))]
	}
	return string(result)
}
```

The `getInt` function parses a provided string to either generate a random integer within a specified range or a random integer between 0 and 9999 if no range is specified, ensuring validity of the input range. Conversely, `randomString` creates a random string of characters with a default length of 10, or a length specified in the input, using `getRandomStrNlen`, which generates a random string of alphanumeric characters of a specified length. These functions collectively facilitate the dynamic generation of data for JSON updates, ensuring variability and accuracy in the generated output.

### Almost there

```bash
run-snapshot-creation :
	gorandomify -t .\events\snapshot_creation_event_template.json -o .\events\snapshot_creation_event.json
	sam build LambdaName
	sam local invoke LambdaName -e .\events\snapshot_creation_event.json
```

Now that the code is ready, I've made it even more convenient to use by building the binary and adding it to my system's path. Additionally, I've updated the `Makefile` of my project to seamlessly integrate our newly created tool. Now, whenever I run the command `make run-snapshot-creation`, it triggers a series of actions.  
Firstly, `gorandomify` references the template file located in `\events\snapshot_creation_event_template.json`, populates the placeholders, and generates the output file in the same directory named `.\events\snapshot_creation_event.json`.  
Subsequently, SAM invokes the lambda with the newly generated event. This streamlined process ensures effortless data generation and lambda invocation, enhancing the efficiency of my project workflow.

```bash
make run-snapshot-creation
```

### Summing it up

In a nutshell, tackling the monotony of repetitive tasks in development and testing became a priority to boost productivity. Enter `gorandomify`, my brainchild, designed to automate JSON data generation for my serverless project. This handy tool eliminates manual data manipulation, letting me focus on the fun stuff. With `gorandomify` seamlessly integrated into my workflow, I've reclaimed precious time and energy, paving the way for smoother sailing in software development. Here's to innovation and problem-solving, making our coding lives a little sweeter, one line of code at a time!
