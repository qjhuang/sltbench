# Intro

Document provides examples for all benchmarks types and describes common
use cases for each type.


# Simple function

Register benchmark for function

```c++
void my_function()
{
	std::vector<size_t> v(100000, 0);
	std::sort(v.begin(), v.end());
}

SLTBENCH_FUNCTION(my_function);

SLTBENCH_MAIN();
```


# Function with many input values

What if `100000` size is not enough,
and we want to benchmark function for several input values?
Well, it is quite easy:

```c++
void my_function(const size_t& count)
{
	std::vector<size_t> vec(count, 0);
	std::sort(vec.begin(), vec.end());
}

static const std::vector<size_t> my_args{ 1024, 2048, 4096 };

SLTBENCH_FUNCTION_WITH_ARGS(my_function, my_args);
```

Requirements:
- function gets input value by const reference
- input values are given as `std::vector`
- operator `std::ostream <<` is provided for input value type (if not, just implement it)


# Function with fixture

What if initialization code like `std::vector<size_t> v(count, 0)`
in the example above should not be benchmarked,
and we are interested in performance of `std::sort` only?
Then, fixture should be used.

```c++
std::vector<size_t> make_my_fixture()
{
	return {100000, 0};
}

void my_sort(std::vector<size_t>& fix)
{
	std::sort(fix.begin(), fix.end());
}
SLTBENCH_FUNCTION_WITH_FIXTURE_BUILDER(my_sort, make_my_fixture);
```

For this case:
* `make_my_fixture` is called per each run, keep its execution time small
especially if function under testing execution time is small (which leads to
huge number of iterations).
* Prefer RAII fixtures or make shure your fixture destructor correctly frees
resources. It is quite obvious for the case of `std::vector`, but it is not for
the case of non-RAII elements like C-style pointer, C-style files etc.

In the example above it is possible to eliminate malloc inside `std::vector`
per each `make_my_fixture` and make fixture generation cheaper. (For functions
with small execution time such trick can dramatically speedup benchmarking
process.) Memory allocated by `std::vector` can be cached between measures:


```c++
class MyFixture
{
public:
	typedef std::vector<size_t> Type;

	MyFixture() {}

	Type& SetUp()
	{
		fixture_.resize(100000, 0);
		return fixture_;
	}

	void TearDown() {}

private:
	Type fixture_;
};

void my_function(MyFixture::Type& fix)
{
	std::sort(fix.begin(), fix.end());
}

SLTBENCH_FUNCTION_WITH_FIXTURE(my_function, MyFixture);
```

Requirements:
- fixture class is default constructible
- fixture has inner typedef `Type`
- fixture has member function `Type& Setup()`
- fixture has member function `void TearDown()`

Be careful, `SetUp` and `TearDown` methods are called per each run.
If `my_function` execution time is small enough (which leads to huge number of
iterations) and `SetUp` and `TearDown` are expensive, benchmark may produce
results for a long long time.


# Function with input values generator

What if input values are not known at compile time?
It is possible to pass input values set from command line or (better) file
with filename given in command line parameters.
Input values generator is designed for this purpose:

```c++
class MyArgsGenerator
{
public:
	typedef size_t ArgType;

	MyArgsGenerator() {}

	std::vector<ArgType> Generate(int argc, char **argv)
	{
		std::vector<ArgType> values;
		// read input values from argc, argv
		// or from file with name given in arc, argv
		// or as you wish...
		return values;
	}
};

void my_function(const MyArgsGenerator::ArgType& arg)
{
	std::vector<size_t> vec(arg, 0);
	std::sort(vec.begin(), vec.end());
}

SLTBENCH_FUNCTION_WITH_ARGS_GENERATOR(my_function, MyArgsGenerator);
```

`Generate` method is called once per function,
all input values returned by generator are copied to internal structure.

Requirements:
- generator is default constructible
- generator has inner typedef `ArgType`
- operator `std::ostream <<` is defined for `ArgType`
- generator has member function `std::vector<ArgType> Generate(int argc, char **argv)`


# Function with fixture and many input values

This example is for the case when we need both
* initialization and
* several input values known at compile time.

Case of simplified fixtures:

```c++
std::vector<std::string> make_fixture(const size_t& arg)
{
	// create and return fixture here
	// ...
}

void my_function(std::vector<std::string>& fix, const size_t& arg)
{
	// code to benchmark here
	// ...
}

static const std::vector<size_t> my_args = { 1024, 2048, 4096 };

SLTBENCH_FUNCTION_WITH_FIXTURE_BUILDER_AND_ARGS(my_function, make_fixture, my_args);
```

Case of class-fixtures:

```c++
class MyFixture
{
public:
	typedef std::vector<size_t> Type;

	MyFixture() {}

	Type& SetUp(const size_t& arg)
	{
		fixture_.resize(arg, 0);
		return fixture_;
	}

	void TearDown() {}

private:
	Type fixture_;
};

void my_function(MyFixture::Type& fix, const size_t& arg)
{
	std::sort(fix.begin(), fix.end());
}

static const std::vector<size_t> my_args = { 1024, 2048, 4096 };

SLTBENCH_FUNCTION_WITH_FIXTURE_AND_ARGS(my_function, MyFixture, my_args);
```


# Function with fixture and input values generator

This example is for the case when we need both:
* initialization and
* several input values known at run time.

Case of simple fixtures:

```c++
class Generator
{
public:
	typedef size_t ArgType;

	Generator() {}

	std::vector<ArgType> Generate(int argc, char **argv)
	{
		std::vector<ArgType> values;
		// init values here ...
		return values;
	}
};

std::vector<std::string> make_fixture(const Generator::ArgType& arg)
{
	// create and return fixture here
	// ...
}

void my_function(std::vector<std::string>& fix, const size_t& arg)
{
	// code to benchmark here
	// ...
}

SLTBENCH_FUNCTION_WITH_FIXTURE_BUILDER_AND_ARGS_GENERATOR(my_function, make_fixture, Generator);
```

Case of class-fixtures:

```c++
class MyArgsGenerator
{
public:
	typedef size_t ArgType;

	MyArgsGenerator() {}

	std::vector<ArgType> Generate(int argc, char **argv)
	{
		std::vector<ArgType> values;
		// init values here ...
		return values;
	}
};

class MyFixture
{
public:
	typedef std::vector<size_t> Type;

	MyFixture() {}

	Type& SetUp(const MyArgsGenerator::ArgType& arg)
	{
		fixture_.resize(arg, 0);
		return fixture_;
	}

	void TearDown() {}

private:
	Type fixture_;
};

void my_function(MyFixture::Type& fix, const MyArgsGenerator::ArgType& arg)
{
	std::sort(fix.begin(), fix.end());
}

SLTBENCH_FUNCTION_WITH_FIXTURE_AND_ARGS_GENERATOR(my_function, MyFixture, MyArgsGenerator);
```


# Function with input values lazy generator

If input value consumes a lot memory and whole values set does not fit into RAM,
lazy generator should be used.

```c++
class Generator
{
public:
	typedef HugeMemoryConsumingStruct ArgType;

	MyArgsGenerator(int argc, char ** argv) { /*...*/ }

	ArgType Generate()
	{
		bool continue_generation = /*...*/;
		if (!continue_generation)
			throw sltbench::StopGenerationException();

		return HugeMemoryConsumingStruct(/*...*/);
	}
};

void my_function(const HugeMemoryConsumingStruct& arg)
{
	// code to benchmark here
	// ...
}

SLTBENCH_FUNCTION_WITH_LAZY_ARGS_GENERATOR(my_function, Generator);
```

Requirements:
- generator is constructible from `(int argc, char** argv)`
- generator has inner typedef `ArgType`
- operator `std::ostream <<` is defined for `ArgType`
- `ArgType` has copy constructor or (better) move constructor
- generator has member function `ArgType Generate()`
- `Generate` member function either returns value for testing
either throws `sltbench::StopGenerationException`


# Function with fixture and input values lazy generator

This example is for the case when we need both:
* initialization and
* lazy arguments generation

Case of simple fixture:

```c++
class Generator
{
public:
	typedef HugeMemoryConsumingStruct ArgType;

	Generator(int argc, char ** argv) { /*...*/ }

	ArgType Generate()
	{
		bool continue_generation = /*...*/;
		if (!continue_generation)
			throw sltbench::StopGenerationException();

		return HugeMemoryConsumingStruct(/*...*/);
	}
};

std::vector<size_t> make_fixture(const Generator::ArgType& arg)
{
	// generate and return fixture
	// ...
}

void my_function(std::vector<size_t>& fix, const HugeMemoryConsumingStruct& arg)
{
	// code to benchmark here
	// ...
}

SLTBENCH_FUNCTION_WITH_FIXTURE_BUILDER_AND_LAZY_ARGS_GENERATOR(my_function, make_fixture, Generator);
```

Case of class-fixtures:

```c++
class Generator
{
public:
	typedef HugeMemoryConsumingStruct ArgType;

	Generator(int argc, char ** argv) { /*...*/ }

	ArgType Generate()
	{
		bool continue_generation = /*...*/;
		if (!continue_generation)
			throw sltbench::StopGenerationException();

		return HugeMemoryConsumingStruct(/*...*/);
	}
};

class Fixture
{
public:
	typedef std::vector<size_t> Type;

	Fixture() {}

	Type& SetUp(const Generator::ArgType& arg)
	{
		/*...*/
		return fixture_;
	}

	void TearDown() {}

private:
	Type fixture_;
};

void my_function(Fixture::Type& fix, const HugeMemoryConsumingStruct& arg)
{
	// code to benchmark
	// ...
}

SLTBENCH_FUNCTION_WITH_FIXTURE_AND_LAZY_ARGS_GENERATOR(my_function, Generator);
```

And the example for simplified fixtures:
