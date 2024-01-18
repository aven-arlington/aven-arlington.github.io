---
layout: page
title: Flashcards Pt.2
date: 2024-01-17 00:00:00-0000
description: Simple CLI utility that quizzes a user on custom clue-answer pairs
img: assets/img/flashcards_cli_thumbnail.png
importance: 2
category: Flashcards
giscus_comments: true
giscus_repo: aven-arlington/flashcards-cli
toc:
  sidebar: left
---
## Topics Covered
- Data deserialization
- Algorithms
- Random number generation

## TLDR
[Try it out in a sandbox...](#try-it-out)\
[Source code for this project post...](https://github.com/aven-arlington/flashcards-cli/tree/project-functionality)\
[Completed Project...](https://github.com/aven-arlington/flashcards-cli)

## Background
This is the second part of a multi-part series that started with [Flashcards Pt.1](/projects/flashcards_pt1/). 

Now that I have a basic layout for the application and a general idea of where I want the different building blocks to go I will start to add functionality. I will start with a configuration file with basic data to test against and deserialize that into usable Rust objects. From there the engine can create a deck of flashcards and shuffle them before presenting them to the user.

## Let's Tackle the Problem
### Defining the Data
Note: I am choosing to use a YAML file format because I prefer the semantics over JSON. This decision is purely subjective and either format can be used with virtually zero changes to the underlying code.

I consider what information a typical flashcard would contain and create a ```struct``` to represent a ```Flashcard``` object. Of course the most important information on a flashcard is the clue-answer pair so fields are created for each. I also intend on the game taking the difficulty of the clue-answer pair into consideration so that the player can get a sense of accomplishment as the questions get harder. I want the configuration to be easy for the user which is in conflict with requiring a difficulty for each card. To accomplish both I will create a ```level``` field to represent the difficulty and make it optional. As discussed in Pt.1, the ```Flashcard``` will need to be public so it can be used everywhere. Finally, I implement a ```new``` function to instantiate them.
```rust
// flashcard.rs
pub struct FlashCard {
    pub clue_side: String,
    pub answer_side: String,
    pub level: Option<u32>,
}

impl FlashCard {
    pub fn new(
        clue_side: String,
        answer_side: String,
        level: Option<u32>,
    ) -> Self {
        Self {
            clue_side,
            answer_side,
            level,
        }
    }
}
```

As features are added to the application, I will need more than a ```Vec<Flashcard>``` to properly hold configuration information. Right now I know that I want to define how many attempts a player can make on a card before the answer is wrong and the correct answer is displayed. So I add these concerns together in a ```Config``` object that also has to be public for the entire application to use.
```rust
// config.rs
pub struct Config {
    pub attempts_before_wrong: usize,
    pub cards: Vec<FlashCard>,
}
```

Now that I have defined what data is required, the YAML file can be generated with some sample data.
```yaml
# flashcards.yaml
attempts_before_wrong: 2

cards:
  - clue_side: Move cursor left
    answer_side: "h"
    level: 0

  - clue_side: Move cursor down
    answer_side: "j"
    level: 0
# ... and so on
```

### Deserialization
Importing the data from the YAML file and into ```Vec<Flashcard>``` is surprisingly easy with the ```serde``` crate doing all the heavy lifting. Adding the ```serde``` crate is done in the ```Cargo.toml``` file like so:
```toml
# Cargo.toml
# Package Declaration...

[dependencies]
serde = { version = "*", features = ["derive"] }
serde_yaml = "*"
```

Both ````Config``` and ```Flashcard``` can be used with ```serde``` simply by deriving the ```Deserialize``` trait.
```rust
// Config used as example. Flashcard would also need to derive Deserialize
#[derive(Deserialize)]
pub struct Config {...}
```

Finally, I am ready to perform the actual data import. This is accomplished with a quick check to be sure that the file exists at the root of the project and returning an error if something goes wrong.
```rust
// config.rs
fn check_default_path() -> Result<PathBuf, &'static str> {
    let mut path_buffer: PathBuf = env::current_dir().unwrap();
    path_buffer.push("flashcards.yaml");
    if path_buffer.try_exists().unwrap() {
        Ok(path_buffer)
    } else {
        Err("The flashcards.yaml configuration file could not be found")
    }
}
```

The resulting ```PathBuf``` is used to read the YAML file into a ```String``` and passed to serde's ```from_str``` function which performs the YAML deserialization and returns a ```Config``` object or an Error. I do a quick check for the basic validity of the ```Config``` object to ensure it should be usable by the game engine.
```rust
// config.rs
pub fn build() -> Result<Config, &'static str> {
    let file_path = Config::check_default_path().unwrap();
    let file_data = fs::read_to_string(file_path).expect("Unable to read config file")
    let conf: Config =
        serde_yaml::from_str(file_data.as_str()).expect("Unable to parse data string")
    assert!(conf.attempts_before_wrong != 0, "Invalid attempts_before_wrong");
    assert!(!conf.cards.is_empty(), "No FlashCards in config file");
    Ok(conf)
}
```

That's it for deserializing the data from the YAML file. Now to create something useful from it.

### Deck Building Algorithm
User supplied data is messy and prone to have errors. In order to build a deck of ```Flashcards``` I have to assume that not all cards will have an assigned difficulty and that if they do, they can be any value, in any order, and non-sequential. These assumptions are tough to work with if allowed to be distributed throughout the codebase. Instead, I want to tackle them in one place, when the ```Deck``` is constructed so that the rest of the application can be assured of working with clean data.

The first step toward clean data is having the ```Vec<Flashcard>``` be sorted by the difficulty level. Adding a few traits to ```Flashcard``` will care of that. I add the ```Eq``` trait to automatically implement the equality operator when comparing ```Flashcard``` objects. Once that is in place, implementations are created for ```Ord```, ```PartialOrd```, and ```PartialEq``` that use the ```level``` field for the comparisons. I also want ```Flashcard``` objects to be easily passed around so I throw in the ```Clone``` trait for good measure.
```rust
// flashcard.rs
#[derive(Deserialize, Default, Debug, Eq, Clone)]
pub struct FlashCard {...}

impl Ord for FlashCard {
    fn cmp(&self, other: &Self) -> Ordering {
        self.level.cmp(&other.level)
    }
}

impl PartialOrd for FlashCard {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for FlashCard {
    fn eq(&self, other: &Self) -> bool {
        self.level == other.level
    }
}
```

Now a ```Vec<Flashcard>``` can be sorted according to the difficulty of the cards so the first thing I do when building a ```Deck``` is exactly that.

I create a ```struct``` that is meant to be a logical representation of a ```Deck``` of cards. Naturally it will contain some type of collection of ```Flashcards``` but which one is best suited for my needs?\
Players will be discouraged if the first few random cards shown to them are all tough. I want the game to ramp up from the easiest (level == 0) cards to more difficult ones so that a player experiences a natural progression of difficulty. After sorting the ```Vec<Flashcard>``` the only assumption that can be made is that the easy ones have a lower index than harder ones. I don't know what distinct ```level``` values were provided by the user, if they are disjointed, or how many cards are in each level. So I will need to separate the cards into distinct levels, re-map the levels from user supplied values into a zero based consecutive range, and then add them to a collection. I chose a ```BTreeMap``` because I want the ```level``` to be an ordered set which I use as a ```key```, and the corresponding value to be a ```Vec<Flashcard>``` that contains all ```Flashcard``` objects with that matching level.\
The ```BTreeMap``` is populated by iterating over the ```Vec<Flashcard>```, checking for a key that corresponds to the cards level with a call to ```entry```, and either inserting the ```Flashcard``` or creating a new ```Vec<Flashcard>``` if none exists.\
I add two helper ```Vec<u32>``` fields to keep track of the available level range after the re-map and the levels that are currently in use by the game.
```rust
// deck.rs
pub struct Deck {
    cards: BTreeMap<u32, Vec<FlashCard>>,
    hand: Vec<FlashCard>,
    available_levels: Vec<u32>,
    current_levels: Vec<u32>,
    rng: ThreadRng,
}

impl Deck {
    pub fn new(mut cards_from_config: Vec<FlashCard>) -> Deck {
        cards_from_config.sort();
        let mut cards: BTreeMap<u32, Vec<FlashCard>> = BTreeMap::new();
        for card in cards_from_config {
            let key = card.level.unwrap_or_default();
            cards
                .entry(key)
                .and_modify(|v| v.push(card.clone()))
                .or_insert(vec![card.clone()]);
        }

        let available_levels: Vec<u32> = cards.keys().cloned().collect();
        let current_levels: Vec<u32> = vec![available_levels[0]];
        let hand = Vec::new();

        Self {
            cards,
            hand,
            available_levels,
            current_levels,
            rng: rand::thread_rng(),
        }
    }
}
```

### Generate a Random Hand of Flashcards
I want the game play to be like a physical game of flashcards where an assistant draws a few cards from the prepared deck, shuffles them, and presents them to the player one by one.\
I create a public ```draw_hand``` function that will update the ```Card::hand``` field with a new ```Vec<Flashcard>``` every round. The game starts at level == 0 but increases with time so all cards with a ```level``` field matching a value contained in ```current_levels``` are cloned into a vector temporarily. I then "shuffle" the cards with a call to ```shuffle``` which performs the operation in place over the cards in ```cards_to_draw_from```.\
I set the hand size to a maximum of 5 arbitrarily but if there are fewer cards than that available the lesser amount is used instead. Then I resize the ```hand``` to contain the cards and clone them over.
```rust
// deck.rs
pub fn draw_hand(&mut self) {
    let mut cards_to_draw_from: Vec<FlashCard> = Vec::new();
    for level in &self.current_levels {
        cards_to_draw_from.append(&mut self.cards.get(level).unwrap().clone());
    }
    cards_to_draw_from.shuffle(&mut self.rng  
    self.hand.clear();
    let hand_size = min(cards_to_draw_from.len(), 5  
    self.hand.resize(hand_size, FlashCard::default());
    self.hand
        .clone_from_slice(&cards_to_draw_from[0..hand_size]);
}
```

I add a few helper functions to the deck for showing the next card, increasing the difficulty, and moving to the next card in the hand. The ```next_card``` and ```hand_count``` functions are pretty trivial and don't require much explaining. The ```increase_level_pool``` function simply scans ```available_levels``` for the first entry that is not contained in ```current_levels``` and adds it to the vector.
```rust
// deck.rs
pub fn next_card(&mut self) -> Option<FlashCard> {
    self.hand.pop()
}
pub fn hand_count(&self) -> usize {
    self.hand.len()
}

pub fn increase_level_pool (&mut self) {
    for level in &self.available_levels {
        if self.current_levels.contains(level) {
            continue;
        }
        println!("Adding level {} to the deck", level);
        self.current_levels.push(*level);
        break;
    }
}
```

### The Game Engine
The "game engine" consists of a public function named ```run``` and two private functions named ```show_card``` and ```get_input``` whose functionality is self evident.\
The ```run``` function contains the main game loop which builds a ```Deck``` of cards as defined in the ```Config```. It then prints a welcome message and instructions before drawing a hand of cards and showing them to the user one by one. When ```deck.next_card``` returns none, the hand is finished and the difficult is increased.
```rust
// engine.rs
pub fn run(config: crate::Config) -> Result<(), Box<dyn Error>> {
    let mut deck = deck::Deck::new(config.cards);

    println!("Welcome to FlashCards!");
    println!("Type \"quit\" to exit the application.");
    loop {
        deck.draw_hand();
        println!("Drawing a fresh hand of {} cards...", deck.hand_count());
        while let Some(card) = deck.next_card() {
            if show_card(&card, config.attempts_before_wrong).is_none() {
                println!("Quitting application");
                return Ok(());
            }
        }
        deck.increase_level_pool();
    }
}
```

The ```show_card``` function displays the ```Flashcard``` clue and then tries to get the answer from the player. If a wrong answer is provided, additional attempts are allowed until the maximum is reached. At that point, the correct answer is supplied to the user and the next card is displayed.
```rust
// engine.rs
fn show_card(card: &crate::FlashCard, mut attempts: usize) -> Option<bool> {
    print!("{}: ", card.clue_side);
    loop {
        match get_input().as_str() {
            "quit" => return None,
            s if s == card.answer_side => {
                println!("Correct!",);
                return Some(true);
            }
            _ => {
                attempts -= 1;
                if attempts >= 1 {
                    println!("Incorrect. Try again");
                    print!("{}: ", card.clue_side);
                } else {
                    println!(
                        "Incorrect. The correct answer was: \"{}\"",
                        card.answer_side
                    );
                    return Some(false);
                }
            }
        }
    }
}
```

Finally the helper function ```get_input``` is used to eliminate some nesting and make the code more readable. It clears the buffer and reads the user's input. That input could be dirty or nonsensical so some minimal sanitization is done and checked for length. If there is no input, a prompt is provided to encourage more input.
```rust
// engine.rs
fn get_input() -> String {
    let mut input = String::new();
    std::io::stdout().flush().expect("Flush failed");
    while std::io::stdin().read_line(&mut input).is_ok() {
        input = input.trim().to_string();
        if !input.is_empty() {
            break;
        }
        input.clear();
        println!("Please provide an answer or \"quit\" to exit the application.");
    }
    input
}
```

That's it! All the pieces are in place to run and play the game.

## Try It Out
I have created a CodeSandbox which follows the content so far and allows readers to try the game out. Once the CodeSandbox is opened, you have the option of signing in and making edits and trying different things. Modifications are automatically saved during context switches from file-to-file or when the Ctrl+s keyboard shortcut is used. Once a change to any of the source files is detected, the ```cargo run``` command will be executed and the game can be played.

[![Edit flashcards-project-functionality](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/p/devbox/flashcards-project-functionality-dtxs89?file=%2Fsrc%2Fmain.rs&embed=1)

## Summary
- The ```serde``` crate makes serialization and deserialization with files very easy to accomplish.
- The ```random``` crate provides an excellent helper function to randomize a slice.
- Working with user supplied data is messy and extra care should be taken to sanitize and validate it.

## Additional Resources
- [Crate rand](https://docs.rs/rand/latest/rand/)
- [Crate serde](https://docs.rs/serde/latest/serde/)
- [Crate serde_yaml](https://docs.rs/serde_yaml/latest/serde_yaml/)
- [Comparing random number generators in Rust](https://blog.logrocket.com/comparing-random-number-generators-rust/)
- [YAML Wrangling with Rust](https://parsiya.net/blog/2022-10-16-yaml-wrangling-with-rust/)
