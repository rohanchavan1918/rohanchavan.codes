+++
title = 'Building wc (word count) in Go'
date = 2024-08-11T18:23:14+05:30
draft = false
+++

When I was not able to find a good alternative to the Linux wc (word count) command for Windows, I decided to build my own version. Written in Go, this tool is not only a faithful port of the original wc but also platform-independent, meaning it works seamlessly on any operating system where Go is supported or you can simply cross compile stand alone binaries for different platforms.

In this blog, I'll walk you through how I built this tool, explaining the design choices, the code structure, and how you can use it.

## Project structure

```
│   go.mod
│   readme.md
├───bin
│       wc_linux_x64
│       wc_win_x64.exe
├───cmd
│       main.go
└───internal
    ├───models
    │       models.go
    ├───utils
    │       utils.go
    └───wc
            counter.go
```

The project is divided into four main components:

- Main Entry Point (cmd/main.go): Handles the command-line interface and parses user input.
- Models (internal/models/models.go): Defines the data structures used across the application.
- Utilities (internal/utils/utils.go): Contains helper functions for flag management and output formatting.
- Word Counter (internal/wc/counter.go): The core logic for processing input and counting words, lines, characters, and bytes.

## Lets Go

Lets first start with writing our main function

The cmd/main.go file is where the program starts. Here, I used the flag package to handle command-line arguments, making it easy for users to specify what they want to count (lines, words, bytes, etc.).

`cmd/main.go`

```go
--- SNIPPED ---
func main() {
	var c, m, l, w bool
	var bytes, chars, lines, words bool

	flag.BoolVar(&c, "c", false, "print the byte counts")
	flag.BoolVar(&bytes, "bytes", false, "print the byte counts")

	flag.BoolVar(&m, "m", false, "print the character counts")
	flag.BoolVar(&chars, "chars", false, "print the character counts")

	flag.BoolVar(&l, "l", false, "print the line counts")
	flag.BoolVar(&lines, "lines", false, "print the line counts")

	flag.BoolVar(&w, "w", false, "print the word counts")
	flag.BoolVar(&words, "words", false, "print the word counts")

	flag.Parse()

	availableCommands := []string{"l", "lines", "w", "words", "c", "bytes", "m", "chars"}
	usedFlags := utils.FlagUsed(availableCommands...)
	input := models.NewInput(usedFlags, flag.Args())
	wc.Process(input, usedFlags)
}
```

You can notice that I have used a seperate flag for short command and long command. Packages like [Cobra](https://github.com/spf13/cobra) does it in more elegant way but i didnt wanted to use any packages for this tool.  
`utils.FlagUsed` is a helper function which returns what flags have been passed by the user in sys args.  
Heres how it looks

```go
func FlagUsed(s ...string) []string {
	usedFlags := []string{}

	for _, fn := range s {
		flag.Visit(func(f *flag.Flag) {
			if f.Name == fn {
				usedFlags = append(usedFlags, fn)
				return
			}
		})
	}

	return usedFlags
}
```

Now that we have the flags passed by the user, lets define a struct which helps us represent data in a better way

```go
type Input struct {
	Bytes bool
	Chars bool
	Lines bool
	Words bool
	Files []string
}

```

The input struct will hold boolean values for different kind of types for which we need the count, and the Files in which we need to count. However, if no flag is passed by user `wc` by default counts the bytes, chars, and new lines, so when we initialize the Input we need to check if the user had passed any flags or not, and accordigly initialize the `Input` struct. Lets create a method to the `Input` struct to handle it.

```go
func NewInput(usedFlags, files []string) Input {

	if len(usedFlags) == 0 {
		// 	Return Input with all as true
		return Input{true, true, true, true, files}
	}

	input := Input{}
	for _, fn := range usedFlags {
		if fn == "c" || fn == "bytes" {
			input.Bytes = true
		}

		if fn == "m" || fn == "chars" {
			input.Chars = true
		}

		if fn == "l" || fn == "lines" {
			input.Lines = true
		}

		if fn == "w" || fn == "words" {
			input.Words = true
		}
	}
	input.Files = files
	return input
}
```

We have what we need to count to process, our main function has a single entry points to process the input. It is defined in the

`internal/wc/counter.go`

```go
func Process(input models.Input, usedFlags []string) {
	// Check if the input is coming from files or stdIn
	if len(input.Files) > 0 {
		ProcessFiles(input, usedFlags)
	} else {
		ProcessStdin(input, usedFlags)
	}

}
```

`wc` can also accept input from the `STDIN` if no files are passed in the input, that is why we have two different functions in the `Process`. Lets have a deeper dive in `ProcessFiles` func

```go
func ProcessFiles(input models.Input, usedFlags []string) error {

	fileResults := []models.FileOp{}
	totalResult := models.Output{}

	for _, file := range input.Files {
		fileResult := models.FileOp{File: file}
		// wc does not give line count, but newLine count thus we need to initialize Lines to -1
		op := models.Output{Lines: -1}
		f, err := os.OpenFile(file, os.O_RDONLY, os.ModePerm)
		if err != nil {
			fmt.Println("Failed to process file ", file)
			fmt.Println("Error  ", err.Error())
			continue
		}
		defer f.Close()

		scanner := bufio.NewScanner(f)
		for scanner.Scan() {
			if err := ProcessLine(input, scanner.Bytes(), &op); err == nil {
				op.AddCount("lines", 1)
			}
		}

		if err := scanner.Err(); err != nil {
			return err
		}

		fileResult.Output = op
		fileResults = append(fileResults, fileResult)
		totalResult.AddCount("bytes", op.Bytes)
		totalResult.AddCount("chars", op.Chars)
		totalResult.AddCount("lines", op.Lines)
		totalResult.AddCount("words", op.Words)
	}

	for _, op := range fileResults {
		utils.PrintResult(usedFlags, op.Output, op.File)
	}
	if len(fileResults) > 1 {
		utils.PrintResult(usedFlags, totalResult, "total")
	}

	return nil
}
```

The `ProcessFiles` function is designed to handle word count operations for multiple files. It begins by initializing an empty slice to store results for each file and an `Output` structure to accumulate the total counts. The function then iterates over each file provided in the input, opening the file in read-only mode. A `bufio.Scanner` is used to read the file line by line, and for each line, the `ProcessLine` function updates the counts for lines, words, characters, and bytes. Since the tool counts newline characters instead of lines, the line count is initialized to `-1`. If any error occurs while reading a file, the function continues with the next file. After processing each file, the individual results are stored, and the totals are updated. Finally, the results for each file are printed, and if multiple files were processed, the combined totals are also displayed. The function returns `nil` if no errors occur during the scanning process.

```go
func ProcessStdin(input models.Input, usedFlags []string) error {
	scanner := bufio.NewScanner(os.Stdin)
	output := models.Output{Lines: -1}

	for scanner.Scan() {
		if err := ProcessLine(input, scanner.Bytes(), &output); err == nil {
			output.AddCount("lines", 1)
		}
	}

	if err := scanner.Err(); err != nil {
		return err
	}
	utils.PrintResult(usedFlags, output, "")
	return nil
}

```

The `ProcessStdin` function is designed to handle word count operations when input is provided via standard input (stdin), making it useful in situations where the tool is used in a pipeline or interactively. The function begins by creating a `bufio.Scanner` to read the input line by line from stdin. It also initializes an `Output` structure with the line count set to `-1` to account for the newline-based line counting. As each line is scanned, the `ProcessLine` function is called to update the counts for lines, words, characters, and bytes. The line count is incremented after each line is successfully processed. Once the scanning is complete, the function checks for any errors that might have occurred during the scanning process. If no errors are found, it prints the results using the `PrintResult` function, displaying the counts based on the flags used. The function returns any scanning errors it encounters, or `nil` if the operation completes successfully.

Below is a quick look for `Output`, this also includes a AddCount method, but we will look into it later on. The `FileOp` struct embeds the Output struct and also includes a File string field which will hold the file name.

```go
type Output struct {
	Bytes int
	Chars int
	Lines int
	Words int
}

type FileOp struct {
	File   string
	Output Output
}
```

If you have observed carefully `ProcessLine` is the function which adds the counts line by line in both the file mode as well as the stdin. lets have a look into it.

```go
func ProcessLine(ip models.Input, line []byte, op *models.Output) error {
	if ip.Bytes {
		op.AddCount("bytes", len(line))
	}

	if ip.Chars {
		op.AddCount("chars", utf8.RuneCount(line))
	}

	if ip.Words {
		// Calculate words
		wSlc := string(line)
		words := 0
		wordStarted := false
		for _, s := range wSlc {
			if string(s) != " " && !wordStarted {
				wordStarted = true
				words++
			} else if string(s) == " " && wordStarted {
				wordStarted = false
			}
		}

		op.AddCount("words", words)
	}

	return nil
}
```

`AddCount` implementation is fairly simple, it increments the count of a particular `field` by the passed amount `n`

```go
func (o *Output) AddCount(field string, n int) {

	switch field {
	case "bytes":
		o.Bytes += n
		return

	case "chars":
		o.Chars += n
		return

	case "lines":
		o.Lines += n
		return

	case "words":
		o.Words += n
		return

	}

}
```

Great! till now we have managed to get the counts, but we need to make sure that the way we print/display our data is similiar to `wc`.

`utils/utils.go`

```go
func PrintFilesResult(flags []string, o []models.FileOp) {
	for _, fn := range o {
		PrintResult(flags, fn.Output, fn.File)
	}
}
```

`PrintFilesResult` will iterate the files and call the PrintResult function

```go
func PrintResult(flags []string, o models.Output, fn string) {
	if len(flags) == 0 {
		fmt.Printf("%d %d %d %s\n", o.Lines, o.Words, o.Bytes, fn)
		return
	}

	op := ""
	for _, f := range flags {
		if f == "l" || f == "lines" {
			op = op + fmt.Sprintf("%d ", o.Lines)
		}

		if f == "w" || f == "words" {
			op = op + fmt.Sprintf("%d ", o.Words)
		}

		if f == "c" || f == "bytes" {
			op = op + fmt.Sprintf("%d ", o.Bytes)
		}

		if f == "m" || f == "chars" {
			op = op + fmt.Sprintf("%d ", o.Chars)
		}

	}

	op = op + fn
	fmt.Println(op)

}
```

If the user havent passed any flags, the default output needs to be printed. we then check for the flags which have been passed and create the final output string.

## Running the code

Lets build the binary

```
go build -o ./bin/wc_win_x64 cmd/main.go
```

Below is the content of `file.txt`

```
hello world
lets go bro
```

we can also run it like

```
PS C:\Users\rohan\Desktop\personal\wc> go run .\cmd\main.go file.txt
1 5 22 file.txt
PS C:\Users\rohan\Desktop\personal\wc>
```

As expected, when we dont pass any flags the default output is printing new lines, number of chars and bytes

Lets pass some flags and test it

```
PS C:\Users\rohan\Desktop\personal\wc> go run .\cmd\main.go -l -w file.txt
1 5 file.txt
PS C:\Users\rohan\Desktop\personal\wc>
```

## Room for improvement

This is a basic implementation, and theres still room for improvement. we could implement go routines etc to make it even faster and would give significant performance benefits when processing large files.

## Give me the code

If you have followed along till here, you definitely want to have a look at the code. Below is the github repo for this proeject.

### [https://github.com/rohanchavan1918/wc](https://github.com/rohanchavan1918/wc)

Thanks and regards!  
Rohan
