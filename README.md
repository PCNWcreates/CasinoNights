#include <Adafruit_NeoPixel.h>
#include <FastLED.h>

// Define pin numbers and variables
#define DICE_PIN 7          // Pin connected to the WS2811B LEDs for dice
#define ROULETTE_PIN 6      // Pin connected to the WS2811B LEDs for roulette
#define BUTTON_PIN 2        // Pin connected to the arcade button
#define NUM_DICE_LEDS 14    // Number of LEDs for dice
#define NUM_ROULETTE_LEDS 12 // Number of LEDs for roulette
#define DEBOUNCE_DELAY 50   // Debounce delay in milliseconds

Adafruit_NeoPixel diceStrip = Adafruit_NeoPixel(NUM_DICE_LEDS, DICE_PIN, NEO_RGB + NEO_KHZ800);
CRGB rouletteLeds[NUM_ROULETTE_LEDS];

// Dice patterns for the 14 LEDs, split into two separate dice
byte dicePatterns1[7][7] = {
  {0, 0, 0, 0, 0, 0, 0}, // 0
  {0, 0, 0, 1, 0, 0, 0}, // 1
  {1, 0, 0, 0, 0, 0, 1}, // 2
  {1, 0, 0, 1, 0, 0, 1}, // 3
  {1, 0, 1, 0, 1, 0, 1}, // 4
  {1, 0, 1, 1, 1, 0, 1}, // 5
  {1, 1, 1, 0, 1, 1, 1}  // 6
};

byte dicePatterns2[7][7] = {
  {0, 0, 0, 0, 0, 0, 0}, // 0
  {0, 0, 0, 1, 0, 0, 0}, // 1
  {1, 0, 0, 0, 0, 0, 1}, // 2
  {1, 0, 0, 1, 0, 0, 1}, // 3
  {1, 0, 1, 0, 1, 0, 1}, // 4
  {1, 0, 1, 1, 1, 0, 1}, // 5
  {1, 1, 1, 0, 1, 1, 1}  // 6
};

// Define the colors of the rainbow for each LED in the dice
CRGB rainbowColors[7] = {
  CRGB::Red,
  CRGB(255, 165, 0), // Adjusted Orange
  CRGB::Yellow,
  CRGB::Green,
  CRGB::Blue,
  CRGB(75, 0, 130),  // Adjusted Indigo
  CRGB(148, 0, 211)  // Adjusted Violet
};

enum State { ATTRACTION_MODE, DICE_ROLL, SHOW_DICE_RESULT, ROULETTE_SPIN, SLOWDOWN, BLINKING, WIN_EFFECT, RE_ROLL };
State currentState = ATTRACTION_MODE;

unsigned long lastDebounceTime = 0;
int currentButtonState = HIGH; // Initialize as HIGH because of pull-up resistor
int lastButtonState = HIGH;    // Initialize as HIGH because of pull-up resistor
int dice1Result = 0;  // Store the result of dice 1
int dice2Result = 0;  // Store the result of dice 2
int stopLED = -1;    // The final LED index the roulette stops on
unsigned long spinStartTime;
int blinkCount = 0;  // Keep track of the number of blinks
int spinCount = 0;   // Keep track of the number of spins

void setup() {
  diceStrip.begin();
  FastLED.addLeds<WS2811, ROULETTE_PIN, GRB>(rouletteLeds, NUM_ROULETTE_LEDS);
  pinMode(BUTTON_PIN, INPUT_PULLUP); // Use internal pull-up resistor
  diceStrip.show(); // Initialize all pixels to 'off'
  FastLED.show();
  randomSeed(analogRead(0)); // Seed random number generator with an analog pin read
}

void loop() {
  // Read the button state with debounce logic
  int reading = digitalRead(BUTTON_PIN);

  if (reading != lastButtonState) {
    lastDebounceTime = millis();  // Reset debounce timer
  }

  if ((millis() - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (reading != currentButtonState) {
      currentButtonState = reading;

      // Button is pressed (LOW when pressed due to pull-up resistor)
      if (currentButtonState == LOW) {
        if (currentState == ATTRACTION_MODE) {
          currentState = DICE_ROLL;
        } else if (currentState == BLINKING && !isMatch()) {
          currentState = RE_ROLL; // Interrupt blinking if no match
        }
      }
    }
  }

  // Save the button state
  lastButtonState = reading;

  switch (currentState) {
    case ATTRACTION_MODE:
      updateAttractionMode();
      break;
    case DICE_ROLL:
      rollDiceEffect();
      if (currentButtonState == HIGH) {
        currentState = SHOW_DICE_RESULT;
        showDiceResult();
      }
      break;
    case SHOW_DICE_RESULT:
      delay(2000); // Show dice result for 2 seconds
      currentState = ROULETTE_SPIN;
      startRouletteSpin();
      break;
    case ROULETTE_SPIN:
      updateRouletteSpin();
      if (millis() - spinStartTime > 3000) { // Spin for 3 seconds
        currentState = SLOWDOWN;
      }
      break;
    case SLOWDOWN:
      startSlowdown();
      checkWinCondition();
      break;
    case BLINKING:
      if (blinkStoppedLED()) {
        currentState = RE_ROLL;  // Interrupt blinking if interrupted
      } else if (blinkCount >= 6) { // Stop after 6 blinks if no match
        currentState = ATTRACTION_MODE;
        updateAttractionMode();
      }
      break;
    case WIN_EFFECT:
      winEffect();
      currentState = ATTRACTION_MODE;
      break;
    case RE_ROLL:
      if (currentButtonState == LOW) {
        currentState = DICE_ROLL;
      }
      break;
  }

  // Continue updating roulette in attraction mode even if dice are rolling
  if (currentState == DICE_ROLL) {
    updateAttractionModeRoulette();
  }
}

// Update attraction mode
void updateAttractionMode() {
  updateAttractionModeDice();
  updateAttractionModeRoulette();
}

// Update attraction mode for dice
void updateAttractionModeDice() {
  for (int i = 0; i < NUM_DICE_LEDS; i++) {
    diceStrip.setPixelColor(i, rainbowColors[i % 7].r, rainbowColors[i % 7].g, rainbowColors[i % 7].b); // Rainbow colors for dice
  }
  diceStrip.show();
}

// Update attraction mode for roulette
void updateAttractionModeRoulette() {
  static unsigned long lastUpdate = 0;
  unsigned long currentTime = millis();
  if (currentTime - lastUpdate >= 100) {
    lastUpdate = currentTime;
    for (int i = 0; i < NUM_ROULETTE_LEDS; i++) {
      rouletteLeds[i] = CHSV(random(255), 255, 255); // Random colors for roulette
    }
    FastLED.show();
  }
}

// Simulate rolling dice (random flashing rainbow colors)
void rollDiceEffect() {
  for (int i = 0; i < NUM_DICE_LEDS; i++) {
    diceStrip.setPixelColor(i, random(0, 256), random(0, 256), random(0, 256)); // Random colors
  }
  diceStrip.show();
  delay(100); // Flash speed
}

// Show final dice result
void showDiceResult() {
  dice1Result = random(0, 7);  // Random number 0-6 for dice 1
  do {
    dice2Result = random(0, 7);  // Random number 0-6 for dice 2
  } while (dice1Result == 0 && dice2Result == 0);  // Ensure both dice do not show 0 at the same time

  // Display dice 1 result (LEDs 1-7)
  for (int i = 0; i < 7; i++) {
    if (dicePatterns1[dice1Result][i] == 1) {
      diceStrip.setPixelColor(i, rainbowColors[i].r, rainbowColors[i].g, rainbowColors[i].b); // Rainbow color
    } else {
      diceStrip.setPixelColor(i, 0, 0, 0); // Turn off LED
    }
  }

  // Display dice 2 result (LEDs 8-14)
  for (int i = 0; i < 7; i++) {
    if (dicePatterns2[dice2Result][i] == 1) {
      diceStrip.setPixelColor(i + 7, rainbowColors[i].r, rainbowColors[i].g, rainbowColors[i].b); // Rainbow color
    } else {
      diceStrip.setPixelColor(i + 7, 0, 0, 0); // Turn off LED
    }
  }
  diceStrip.show();
}

// Start Roulette spin
void startRouletteSpin() {
  spinStartTime = millis();
  spinCount++; // Increment the spin count
}

// Update Roulette spin
void updateRouletteSpin() {
  unsigned long currentTime = millis();
  int ledIndex = ((currentTime - spinStartTime) * 20 / 100) % NUM_ROULETTE_LEDS;
  updateRouletteLEDs(ledIndex);
  FastLED.show();
}

// Start slowdown effect
void startSlowdown() {
  unsigned long duration = random(3000, 5000);
  unsigned long endTime = millis() + duration;
  unsigned long startTime = millis();
  unsigned long slowdownDuration = 5000; // Duration of slowdown phase
  unsigned long lastUpdateTime = millis();
  int ledIndex = 0;
  int delayTime = 10;

  while (millis() < endTime) {
    unsigned long currentTime = millis();
    if (currentTime - lastUpdateTime >= delayTime) {
      lastUpdateTime = currentTime;
      updateRouletteLEDs(ledIndex);
      FastLED.show();
      ledIndex = (ledIndex + 1) % NUM_ROULETTE_LEDS;
    }
    delay(10);
  }

  // Slowdown phase: gradually increase delay
  while (millis() - startTime < slowdownDuration) {
    unsigned long currentTime = millis();
    if (currentTime - lastUpdateTime >= delayTime) {
      lastUpdateTime = currentTime;
      updateRouletteLEDs(ledIndex);
      FastLED.show();
      ledIndex = (ledIndex + 1) % NUM_ROULETTE_LEDS;
      delayTime = map(currentTime - startTime, 0, slowdownDuration, 10, 200); // Increase delay from 10ms to 200ms
    }
  }

  stopLED = ledIndex; // Ensure it stops on the current LED
  FastLED.clear(); // Clear all LEDs before flashing the stopped LED
  flashStoppedLED(stopLED, 3);
  populateWheelWithRainbow();
  startFadingEffect();
}

// Check win condition
void checkWinCondition() {
  // Adjust stopLED to match the numbering from 1-12
  int adjustedStopLED = (stopLED + 1) % NUM_ROULETTE_LEDS;
  if (adjustedStopLED == 0) {
    adjustedStopLED = 1;  // Treat 0 as 1
  }

  // Combine the results of both dice, treating 0 as 1
  int combinedResult = (dice1Result == 0 ? 1 : dice1Result) + (dice2Result == 0 ? 1 : dice2Result);

  if (spinCount % 50 == 0) {
    currentState = WIN_EFFECT; // Force a win every 50 spins
  } else if (adjustedStopLED == combinedResult) {
    currentState = WIN_EFFECT;
  } else {
    showDiceRed();
    blinkCount = 0; // Reset blink count
    currentState = BLINKING; // Move to blinking if no match
  }
}

// Flash the stopped LED multiple times with a random vibrant color
void flashStoppedLED(int ledIndex, int flashes) {
  for (int i = 0; i < flashes; i++) {
    CRGB vibrantColor = CHSV(random(255), 255, 255); // Random vibrant color
    rouletteLeds[ledIndex] = vibrantColor;
    FastLED.show();
    delay(200);
    rouletteLeds[ledIndex] = CRGB::Black;
    FastLED.show();
    delay(200);
  }
}

// Populate the entire wheel with solid ROYGBIV colors
void populateWheelWithRainbow() {
  for (int i = 0; i < NUM_ROULETTE_LEDS; i++) {
    rouletteLeds[i] = rainbowColors[i % 7];
  }
  FastLED.show();
}

// Start fading effect for the stopped LED
void startFadingEffect() {
  unsigned long startTime = millis();
  while (millis() - startTime < 5000) { // Run the effect for 5 seconds
    for (int hue = 0; hue < 255; hue++) {
      rouletteLeds[stopLED] = CHSV(hue, 255, 255); // Change hue for full spectrum effect
      FastLED.show();
      delay(20); // Adjust delay for smooth fading
    }
  }
}

// Blink stopped LED and return true if interrupted
bool blinkStoppedLED() {
  static unsigned long lastBlinkTime = 0;

  if (millis() - lastBlinkTime >= 1000) { // Blink every 1 second
    lastBlinkTime = millis();
    blinkCount++;

    if (blinkCount > 6) { // Stop after 6 blinks
      currentState = ATTRACTION_MODE; // Reset to attraction mode
      return false;
    }

    rouletteLeds[stopLED] = (blinkCount % 2 == 0) ? CRGB(CHSV(random(255), 255, 255)) : CRGB::Black;
    FastLED.show();
  }

  return (currentButtonState == LOW); // Return true if interrupted by button press
}

// Win effect
void winEffect() {
  // Pulse the winning LED through the full spectrum
  unsigned long pulseStartTime = millis();
  CRGB winColor = CHSV(random(255), 255, 255); // Random color for win effect

  while (millis() - pulseStartTime < 5000) {
    bool blinkState = ((millis() / 500) % 2) == 0; // Blink every 500ms
    CRGB color = blinkState ? winColor : CRGB::Black;

    rouletteLeds[stopLED] = color;
    FastLED.show();

    // Match dice to roulette win mode
    for (int i = 0; i < 7; i++) {
      if (dicePatterns1[dice1Result - 1][i] == 1) {
        diceStrip.setPixelColor(i, color.r, color.g, color.b); // Apply winning color
      } else {
        diceStrip.setPixelColor(i, 0, 0, 0); // Turn off LED
      }
      if (dicePatterns2[dice2Result - 1][i] == 1) {
        diceStrip.setPixelColor(i + 7, color.r, color.g, color.b); // Apply winning color
      } else {
        diceStrip.setPixelColor(i + 7, 0, 0, 0); // Turn off LED
      }
    }
    diceStrip.show();

    delay(10);
  }
}

// Update LEDs with clockwise pattern
void updateRouletteLEDs(int index) {
  FastLED.clear();
  if (index >= 0 && index < NUM_ROULETTE_LEDS) {
    rouletteLeds[index] = CHSV(random(255), 255, 255); // Random color for the selected LED
  }
  FastLED.show();
}

// Show dice red (default behavior)
void showDiceRed() {
  for (int i = 0; i < 7; i++) {
    if (dicePatterns1[dice1Result - 1][i] == 1) {
      diceStrip.setPixelColor(i, rainbowColors[i].r, rainbowColors[i].g, rainbowColors[i].b); // Rainbow color for dice 1
    } else {
      diceStrip.setPixelColor(i, 0, 0, 0); // Turn off LED for dice 1
    }
    if (dicePatterns2[dice2Result - 1][i] == 1) {
      diceStrip.setPixelColor(i + 7, rainbowColors[i].r, rainbowColors[i].g, rainbowColors[i].b); // Rainbow color for dice 2
    } else {
      diceStrip.setPixelColor(i + 7, 0, 0, 0); // Turn off LED for dice 2
    }
  }
  diceStrip.show();
}

// Check if there is a match
bool isMatch() {
  // Combine the results of both dice
  int combinedResult = dice1Result + dice2Result;

  // Adjust stopLED to match the numbering from 1-12
  return ((stopLED + 1) % NUM_ROULETTE_LEDS == combinedResult);
}