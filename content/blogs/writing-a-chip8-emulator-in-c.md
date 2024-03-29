---
title: "Writing a CHIP8 Emulator in C"
date: 2024-03-02T19:28:44+05:30
---

## Storytime
I was always interested in systems programming and one day my friend told me about emulator development and introduced me to CHIP8 and i was fascinated by it, so i wrote [my implementation of CHIP8](https://github.com/suryapsp/chip8/).

## What is CHIP8?
CHIP8 is a interpreted programming language which was first used in COSMAC VIP microcomputer. CHIP8 is not an actual hardware gaming console like Playstation, NES, SNES and Gameboy. CHIP8 programs are basically instructions which can decoded by an interpreter. CHIP8 programs can run on any computer as long as they have that interpreter. In this tutorial we are going to create an interpreter like that.

## Prerequsite
I expect clear understanding of C programming language specially topics like conditionals, loops, pointers, file handling and sturctures.

You will also require basic understanding of binary and hexadecimal number systems and understanding of some library by which you can do things like drawing graphics, keyboard input, etc. we are going to use [SDL2](https://www.libsdl.org/) in this tutorial.

## CHIP8 Specifications

- Memory
    - CHIP8 has access to 4 KB of RAM.

- Keyboard
    - Original CHIP8 has 16 keyboard keys which match to first 16 hexadecimal numbers 0 to F.
    - This Keyboard layout will be used.
    ```markdown
        CHIP8         KEYBOARD

        1 2 3 C       1 2 3 4
        4 5 6 D       Q W E R
        7 8 9 E       A S D F
        A 0 B F       Z X C V
    ```
- Display
    - CHIP8 used 64x32 pixels monochrome(only black and white) display.

- Timers
    - It has an 8 bit delay timer which decrements at a rate of 60Hz(60 times per second) until it reaches zero.
    - It has an 8 bit sound timer which decrements at a rate of 60Hz(60 times per second) unitl it reaches zero.

- 16 8-bit registers
    - CHIP8 has 16 8 bit general purpose Registers V0 to VF.
    - VF is special becuse it is used as a flag register. Many programs set it to either 1 or 0.

- 16 bit program counter
    - program counter usually referred as PC is used to store the address of next instruction to execute.

- 16 bit index register
    - index register or I is a special register used to store memory address.

- 16 level stack
    - CHIP8 has 16 levels of stack which is used to call functions and return from them.

### CHIP8 structure
CHIP8 struct will contation all the above mentioned things
```c
typedef enum {
	QUIT,
	RUNNING,
	PAUSED
} emulator_state_t;

typedef struct{
	uint16_t opcode;
	uint16_t NNN;
	uint8_t NN;
	uint8_t N;
	uint8_t X;
	uint8_t Y;
}instruction_t;

// Main CHIP8 Obj
typedef struct{
	emulator_state_t state;
	uint8_t ram[4096];
	bool display[64*32]; 
	uint16_t stack[12]; 
	uint16_t *SP;
	uint8_t V[16]; 
	uint16_t I; 
	uint8_t delay_timer; 
	uint8_t sound_timer; 
	bool keypad[16]; 
	const char *rom_name; 
	uint16_t PC; 
	instruction_t inst;
	bool draw; 
} chip8_t;
```
Here ```instruction_t inst``` is a struct which contains opcode and some other things which will be explained later in the tutorial.


## Initializing CHIP8
Before we could execute the instructions we would have to first load the ROM and Fonts into the CHIP8 RAM.

```c
bool init_chip8(chip8_t *chip8, const char rom_name[]){
	const uint32_t entry_point = 0x200;
	const uint8_t font[] = {
		0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
    	0x20, 0x60, 0x20, 0x20, 0x70, // 1
    	0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
    	0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
    	0x90, 0x90, 0xF0, 0x10, 0x10, // 4
    	0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
    	0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
    	0xF0, 0x10, 0x20, 0x40, 0x40, // 7
    	0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
    	0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
    	0xF0, 0x90, 0xF0, 0x90, 0x90, // A
    	0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
    	0xF0, 0x80, 0x80, 0x80, 0xF0, // C
    	0xE0, 0x90, 0x90, 0x90, 0xE0, // D
    	0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
    	0xF0, 0x80, 0xF0, 0x80, 0x80  // F
	};

	// Initialize chip8 machine
	memset(chip8, 0, sizeof(chip8_t));

	// Load Font
	memcpy(&chip8 -> ram[0], font, sizeof(font));

	// Open ROM
	FILE *rom = fopen(rom_name, "rb");
	if(!rom){
		SDL_Log("ROM FILE is Invalid\n");
		return false;
	}


	// ROM Size
	fseek(rom, 0, SEEK_END);
	const size_t rom_size = ftell(rom);
	const size_t max_size = sizeof chip8->ram - entry_point;
	rewind(rom);

	if(rom_size > max_size){
		SDL_Log("ROM FILE is Too Large to Handle\n");
		return false;
	}

	// Read ROM
	if(fread(&chip8->ram[entry_point], rom_size, 1, rom) != 1){
		SDL_Log("Can't Read the ROM \n");
		return false;
	};

	// Close ROM
	fclose(rom);

	// Defaults
	chip8 -> state = RUNNING;
	chip8 -> PC = entry_point;
	chip8 -> rom_name = rom_name;
	chip8 -> SP = chip8->stack;

	return true; //Sucess
}
```

## SDL and Configs
Before doing anything else lets first initialize SDL and add config struct to manager things like fg color and bg color. In future, we can make command line arguments to give options to user to change these things if they want but we are not going to do that in this tutorial.
```c
typedef struct {
	uint32_t window_width;
	uint32_t window_height;
	uint32_t foreground_color; // foreground color R-8 G-8 B-8 A-8
	uint32_t background_color; // bg color R-8 G-8 B-8 A-8

	uint32_t scale_factor; // Scale CHIP-8 px

	bool pixel_outlines; // does the user want pixel outlines or not

	uint32_t inst_per_sec; // cpu clock rate

	uint32_t square_wave_freq;  //frequency of square wave sound

	uint32_t audio_sample_rate;

	int16_t volume;
}config_t;

typedef struct 
{
	SDL_Window *window;
	SDL_Renderer *renderer;
	SDL_AudioSpec want, have;
	SDL_AudioDeviceID dev;
}sdl_t;

bool init_sdl(sdl_t *sdl, config_t *config){
	if(SDL_Init(SDL_INIT_VIDEO|SDL_INIT_AUDIO|SDL_INIT_TIMER) != 0){
		SDL_Log("Can't Initialize SDL Subsystem %s \n", SDL_GetError());
		return false; // Initialization Failed
	}


	sdl -> window = SDL_CreateWindow("CHIP-8 Emulator", SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED, config->window_width * config->scale_factor, config->window_height * config->scale_factor, 0);

	if(!sdl -> window){
		SDL_Log("could not create window %s\n", SDL_GetError());
		return false;
	}

	sdl -> renderer = SDL_CreateRenderer(sdl -> window, -1, SDL_RENDERER_ACCELERATED);

	if(!sdl->renderer){
		SDL_Log("could not create renderer %s\n", SDL_GetError());
		return false;
	}

	sdl->want = (SDL_AudioSpec){
		.freq = 44100,
		.format = AUDIO_S16LSB, //signed 16 bit little indian
		.channels = 1, //mono 1 channel
		.samples = 512,
		.callback = audio_callback,
		.userdata = config
	};

	sdl->dev = SDL_OpenAudioDevice(NULL,0, &sdl->want, &sdl->have, 0);

	if(sdl->dev == 0){
		SDL_Log("could not get any audio device %s\n", SDL_GetError());
		return false;
	}

	if((sdl->want.format != sdl->have.format)||(sdl->want.channels != sdl->have.channels)){
		SDL_Log("could not get desired audio spec\n");
		return false;
	}

	return true; // Initialization Done
}

// Initial Emulator Config from actual args
bool set_config(config_t *config, int argc, char **argv){
	// Defaults
	*config = (config_t){
		.window_width = 64,
		.window_height = 32, //OG CHIP8 Resolution
		.foreground_color = 0xFFFFFFFF, //White 
		.background_color = 0x000000FF,  //Yellow
		.scale_factor = 20, // Scale 20x
		.pixel_outlines = true, // Draw pixel outlines
		.inst_per_sec = 700, // Default Clock Rate
		.square_wave_freq = 440, 
		.audio_sample_rate = 44100,
		.volume = 3000,
	};

	// Change Defaults
	for(int i = 0; i < argc; i++){
		(void)argv[i];
	}

	return true;

    void final_cleanup(sdl_t sdl){
	    SDL_DestroyRenderer(sdl.renderer);
	    SDL_DestroyWindow(sdl.window);
	    SDL_CloseAudioDevice(sdl.dev);
	    SDL_Quit(); // Quit SDL Subsystem
    }

    void update_timers(const sdl_t sdl, chip8_t *chip8){
	if(chip8->delay_timer > 0){
		chip8->delay_timer--;
	}

	if(chip8->sound_timer > 0){
		chip8->sound_timer--;
		SDL_PauseAudioDevice(sdl.dev, 0); // Play Audio
	}
	else{
		SDL_PauseAudioDevice(sdl.dev, 1); //Pause Audio
	}
}
}
```

## Screen and Sound

```c
// Clear Screen
void clear_screen(const sdl_t sdl, const config_t config){
	const uint8_t r = (config.background_color >> 24) & 0xFF;
	const uint8_t g = (config.background_color >> 16) & 0xFF;
	const uint8_t b = (config.background_color >> 8) & 0xFF;
	const uint8_t a = (config.background_color >> 0) & 0xFF;
	SDL_SetRenderDrawColor(sdl.renderer, r,g,b,a);
	SDL_RenderClear(sdl.renderer);
}

// Update window changes
void update_screen(const sdl_t sdl, config_t config, chip8_t chip8){
	SDL_Rect rect = {.x = 0, .y = 0, .w = config.scale_factor, .h = config.scale_factor};

// Color Values
	const uint8_t bg_r = (config.background_color >> 24) & 0xFF;
	const uint8_t bg_g = (config.background_color >> 16) & 0xFF;
	const uint8_t bg_b = (config.background_color >> 8) & 0xFF;
	const uint8_t bg_a = (config.background_color >> 0) & 0xFF;

	const uint8_t fg_r = (config.foreground_color >> 24) & 0xFF;
	const uint8_t fg_g = (config.foreground_color >> 16) & 0xFF;
	const uint8_t fg_b = (config.foreground_color >> 8) & 0xFF;
	const uint8_t fg_a = (config.foreground_color >> 0) & 0xFF;



// Draw Rectange per pixel
	for(uint32_t i = 0; i < sizeof(chip8.display); i++){
		// Translate i value to x y coordinates
		rect.x = (i % config.window_width)*config.scale_factor;
		rect.y = (i / config.window_width) *config.scale_factor;


		if(chip8.display[i]){
			// draw foreground
			SDL_SetRenderDrawColor(sdl.renderer, fg_r, fg_g, fg_b, fg_a);
			SDL_RenderFillRect(sdl.renderer, &rect);

			// pixel outlines not necessary can be used as user option
			if(config.pixel_outlines){
				SDL_SetRenderDrawColor(sdl.renderer, bg_r, bg_g, bg_b, bg_a);
				SDL_RenderDrawRect(sdl.renderer, &rect);
			}


		}
		else{
			// draw background
			SDL_SetRenderDrawColor(sdl.renderer, bg_r, bg_g, bg_b, bg_a);
			SDL_RenderFillRect(sdl.renderer, &rect);
		}
	}



	SDL_RenderPresent(sdl.renderer);
}

void audio_callback(void *userdata, uint8_t *stream, int len){
    config_t *config = (config_t *) userdata;

    int16_t *audio_data = (int16_t *) stream;

    static uint32_t running_sample_index = 0;

    const int32_t sqaure_wave_period = config->audio_sample_rate / config->square_wave_freq;
    const int32_t half_sqaure_wave_period = sqaure_wave_period/2;

    for(int i = 0; i < len/2; i++){
        audio_data[i] = ((running_sample_index++ / half_sqaure_wave_period) % 2) ? 
        config->volume : 
        -config->volume;
    }
}
```

## Taking Keyboard Input

Without user input our CHIP8 is incomplete.
We are going to use the keymaping below.

```markdown
CHIP-8 KEYPAD		QWERTY

1 2 3 C 			1 2 3 4
4 5 6 D				Q W E R
7 8 9 E				A S D F
A 0 B F				Z X C V

Extra Keys
ESCAPE              QUIT
SPACE               PAUSE
BACKSPACE           RESTART
```

We are going handle user input by creating a handle_input function which uses SDL to handle user input.

```c
void handle_input(chip8_t *chip8){
	SDL_Event event;

	while(SDL_PollEvent(&event)){
		switch (event.type) {
			case SDL_QUIT:
				chip8->state = QUIT;
				return ;

			case SDL_KEYDOWN:
				switch(event.key.keysym.sym){
					case SDLK_ESCAPE:
						chip8 -> state = QUIT;
						return;

					case SDLK_SPACE:
						if(chip8->state == RUNNING){
							chip8->state = PAUSED; // pause
							puts("paused"); 
						}
						else{
							chip8->state = RUNNING; // resume
						}
						break;

					case SDLK_BACKSPACE:
						init_chip8(chip8, chip8->rom_name);
						break;

					case SDLK_1:
						chip8->keypad[0x1] =  true;
						break;

					case SDLK_2:
						chip8->keypad[0x2] =  true;
						break;

					case SDLK_3:
						chip8->keypad[0x3] =  true;
						break;

					case SDLK_4:
						chip8->keypad[0xC] =  true;
						break;

					case SDLK_q:
						chip8->keypad[0x4] =  true;
						break;

					case SDLK_w:
						chip8->keypad[0x5] =  true;
						break;

					case SDLK_e:
						chip8->keypad[0x6] =  true;
						break;

					case SDLK_r:
						chip8->keypad[0xD] =  true;
						break;

					case SDLK_a:
						chip8->keypad[0x7] =  true;
						break;

					case SDLK_s:
						chip8->keypad[0x8] =  true;
						break;

					case SDLK_d:
						chip8->keypad[0x9] =  true;
						break;

					case SDLK_f:
						chip8->keypad[0xE] =  true;
						break;

					case SDLK_z:
						chip8->keypad[0xA] =  true;
						break;

					case SDLK_x:
						chip8->keypad[0x0] =  true;
						break;

					case SDLK_c:
						chip8->keypad[0xB] =  true;
						break;

					case SDLK_v:
						chip8->keypad[0xF] =  true;
						break;


					default:
						break;

				}
				break;

			case SDL_KEYUP:
				switch(event.key.keysym.sym){

					case SDLK_1:
						chip8->keypad[0x1] =  false;
						break;

					case SDLK_2:
						chip8->keypad[0x2] =  false;
						break;

					case SDLK_3:
						chip8->keypad[0x3] =  false;
						break;

					case SDLK_4:
						chip8->keypad[0xC] =  false;
						break;

					case SDLK_q:
						chip8->keypad[0x4] =  false;
						break;

					case SDLK_w:
						chip8->keypad[0x5] =  false;
						break;

					case SDLK_e:
						chip8->keypad[0x6] =  false;
						break;

					case SDLK_r:
						chip8->keypad[0xD] =  false;
						break;

					case SDLK_a:
						chip8->keypad[0x7] =  false;
						break;

					case SDLK_s:
						chip8->keypad[0x8] =  false;
						break;

					case SDLK_d:
						chip8->keypad[0x9] =  false;
						break;

					case SDLK_f:
						chip8->keypad[0xE] =  false;
						break;

					case SDLK_z:
						chip8->keypad[0xA] =  false;
						break;

					case SDLK_x:
						chip8->keypad[0x0] =  false;
						break;

					case SDLK_c:
						chip8->keypad[0xB] =  false;
						break;

					case SDLK_v:
						chip8->keypad[0xF] =  false;
						break;


					default:
						break;

				}
				break;

			default:
				break;
		}
	}
}
```

## Emulating Instructions
Ok so this is the fun part. Now we are going to emulate all the instructions that are present in CHIP8.

But before that we are going to dissect instruction_t.

```c
typedef struct{
	uint16_t opcode;
	uint16_t NNN;
	uint8_t NN;
	uint8_t N;
	uint8_t X;
	uint8_t Y;
}instruction_t;
```

Opcodes >> These are part of the instruction which specifies which operation to perform. There are 35 opcodes in CHIP8 which are stored in big endian format and are all two bytes long. 
In big indian the LSB(Least Significant Bit) is stored at the highest memory address.

NNN >> It stored the address of a memory location.

NN >> It is a 8 bit constant.

X & Y >> 4 bit register identifier.

Ok so we will emulate all the instructions in emulate_instruction function.

```c
void emulate_instructions(chip8_t *chip8, const config_t config){
	// Get opcode from RAM
	chip8->inst.opcode = (chip8->ram[chip8->PC] << 8) | chip8 -> ram[chip8->PC+1];
	chip8->PC +=2;

	// SYMBOLS 
	chip8->inst.NNN = chip8->inst.opcode & 0x0FFF;
	chip8->inst.NN = chip8->inst.opcode & 0x0FF;
	chip8->inst.N = chip8->inst.opcode & 0x0F;
	chip8->inst.X = (chip8->inst.opcode >> 8) & 0x0F;
	chip8->inst.Y = (chip8->inst.opcode >> 4) & 0x0F;

// #ifdef DEBUG
// 	print_debug_output(chip8);
// #endif
	// EMULATING OPCODES INSTRUCTIONS
	switch((chip8->inst.opcode >> 12) & 0x0F){

		case 0x00:
			if(chip8->inst.NN == 0xE0){
				// 0x00E0 Display Clear
				memset(chip8->display, false, sizeof(chip8->display));
				chip8->draw = true;
			}
			else if(chip8->inst.NN == 0xEE){
				// Returns from a subroutine
				// 0x00EE
				/*
				Set Program Counter to last address of function(subroutine) call (pop it off the stack)
				*/
				chip8->PC = *--chip8->SP;

			}
			break;

		case 0x01:
			// 1NNN

			// Jumps to address NNN

			chip8->PC = chip8->inst.NNN;
			break;

		case 0x02:
			// Calls subroutine at NNN
			// 0x2NNN
			/*
			Store Current Address from the program counter to the stack (PUSH IT TO THE STACK)
			Set the program counter to NNN 
			*/
			*chip8->SP++ = chip8->PC; 
			chip8->PC = chip8->inst.NNN;
			break;

		case 0x03:
			// 0x3XNN
			// Skips the next instruction if VX equals NN (usually the next instruction is a jump to skip a code block)

			if(chip8->V[chip8->inst.X] == chip8->inst.NN){
				chip8->PC += 2;
			}

			break;

		case 0x04:
			// 0x4XNN
			// Skips the next instruction if VX does not equal NN (usually the next instruction is a jump to skip a code block).
			// Opposite of 0x3XNN

			if(chip8->V[chip8->inst.X] != chip8->inst.NN){
				chip8->PC += 2;
			}

			break;

		case 0x05:
			// 0x5XY0
			// Skips the next instruction if VX equals VY (usually the next instruction is a jump to skip a code block)

			if(chip8->inst.N != 0){
				break; // wrong opcode
			}

			if(chip8->V[chip8->inst.X] == chip8->V[chip8->inst.Y]){
				chip8->PC += 2;
			}

			break;

		case 0x06:
			// Sets VX to NN
			// 0x6XNN

			chip8->V[chip8->inst.X] = chip8->inst.NN;

			break;

		case 0x07:
			// Adds NN to VX (carry flag is not changed)
			// 0x7XNN
			chip8->V[chip8->inst.X] += chip8->inst.NN;
			break;

		case 0x08:
			switch(chip8->inst.N){
				case 0:
					// 0x8XY0
					// Sets VX to the value of VY

					chip8->V[chip8->inst.X] = chip8->V[chip8->inst.Y];

					break;

				case 1:
					// 0x8XY1
					// Sets VX to VX or VY (bitwise OR operation)

					chip8->V[chip8->inst.X] |= chip8->V[chip8->inst.Y];

					break;

				case 2:
					// 0x8XY2
					// Sets VX to VX and VY (bitwise AND operation)

					chip8->V[chip8->inst.X] &= chip8->V[chip8->inst.Y];

					break;

				case 3:
					// 0x8XY3
					// Sets VX to VX xor VY

					chip8->V[chip8->inst.X] ^= chip8->V[chip8->inst.Y];

					break;

				case 4:
					// 0x8XY4
					// Adds VY to VX
					// VF is set to 1 when there's an overflow, and to 0 when there is not

					if((uint16_t)(chip8->V[chip8->inst.X] + chip8->V[chip8->inst.Y]) > 255){
						chip8->V[0xF] = 1;
					}

					chip8->V[chip8->inst.X] += chip8->V[chip8->inst.Y];

					break;

				case 5:
					// 0x8XY5
					// VY is subtracted from VX
					// VF is set to 0 when there's an underflow, and 1 when there is not. (i.e. VF set to 1 if VX >= VY and 0 if not). 

					if((int16_t)(chip8->V[chip8->inst.X] - chip8->V[chip8->inst.Y]) < 0){
						chip8->V[0xF] = 0;
					}
					else{
						chip8->V[0xF] = 1;
					}

					chip8->V[chip8->inst.X] -= chip8->V[chip8->inst.Y];

					break;					

				case 6:
					// 0X8XY6
					// Stores the least significant bit of VX in VF and then shifts VX to the right by 1

					chip8->V[0xF] = chip8->V[chip8->inst.X] & 1;

					chip8->V[chip8->inst.X] >>= 1;

					break;

				case 7:
					// 0x8XY7
					// Sets VX to VY minus VX. VF is set to 0 when there's an underflow, and 1 when there is not. (i.e. VF set to 1 if VY >= VX)

					if((int16_t) (chip8->V[chip8->inst.Y] -  chip8->V[chip8->inst.X]) < 0){
						chip8->V[0xF] = 0;
					}
					else{
						chip8->V[0xF] = 1;
					}

					chip8->V[chip8->inst.X] = chip8->V[chip8->inst.Y] - chip8->V[chip8->inst.X];

					break;

				case 0xE:
					// 0x8XYE
					// Stores the most significant bit of VX in VF and then shifts VX to the left by 1

					chip8->V[0xF] = (chip8->V[chip8->inst.X] & 0x80) >> 7;

					chip8->V[chip8->inst.X] <<= 1;

					break;


				default:
					break;
			}
			break;

		case 0x09:
			// 0x9XY0
			// Skips the next instruction if VX does not equal VY. (Usually the next instruction is a jump to skip a code block)

			if(chip8->V[chip8->inst.X] != chip8->V[chip8->inst.Y]){
				chip8->PC += 2;
			}

			break;

		case 0x0A:
			// 0xANNN
			// Sets I to the address NNN

			chip8->I = chip8->inst.NNN;
			break;

		case 0x0B:
			// 0xBNNN
			// Jumps to the address NNN plus V0

			chip8->PC = chip8->inst.NNN + chip8->V[0];

			break;

		case 0x0C:
			// 0xCXNN
			// Sets VX to the result of a bitwise and operation on a random number (Typically: 0 to 255) and NN
			chip8->V[chip8->inst.X] = (rand() % 256) & chip8->inst.NN;

			break;


		case 0x0D:
			/*
			Draws a sprite at coordinate (VX, VY) that has a width of 8 pixels and a height of N pixels. Each row of 8 pixels is read as bit-coded starting from memory location I; I value does not change after the execution of this instruction. As described above, VF is set to 1 if any screen pixels are flipped from set to unset when the sprite is drawn, and to 0 if that does not happen
			*/
			// 0xDXYN

			uint8_t x = chip8->V[chip8->inst.X];
			uint8_t y = chip8->V[chip8->inst.Y];

			uint8_t og_x = x; // original x value

			uint8_t height = chip8->inst.N;

			// wrap the coordinates if they are bigger than the screen size
			x %= config.window_width;
			y %= config.window_height;

			// Set carry/collision flag to 0
			chip8->V[0xF] = 0;

			// Loop to iterate over N rows in the sprite
			for(uint8_t i = 0; i < height; i++){
				uint8_t sprite_data = chip8->ram[chip8->I + i];

				x = og_x; //reset x for next row
				// Loop to iterate over each bit(pixel) in the sprite
				for(int8_t j = 7; j >= 0; j--){
					bool *pixel = &chip8 -> display[y*config.window_width + x];

					bool sprite_bit = (sprite_data&(1<<j));

					if(sprite_bit && *pixel){
						chip8->V[0xF] = 1;
					}

					// XOR display pixel with sprite pixel to set it on or off
					*pixel ^= sprite_bit;


					// 
					if(++x >= config.window_width) break;
				}
				// 
				if(++y >= config.window_height) break;
			}
			chip8->draw = true;
			break;


		case 0xE:
			switch(chip8->inst.NN){
				case 0x9E:
					// 0xEX9E
					// Skips the next instruction if the key stored in VX is pressed (usually the next instruction is a jump to skip a code block)

					if(chip8->keypad[chip8->V[chip8->inst.X]] == true){
						chip8->PC += 2;
					}

					break;

				case 0xA1:
					// 0xEXA1
					// Skips the next instruction if the key stored in VX is not pressed (usually the next instruction is a jump to skip a code block)

					if(chip8->keypad[chip8->V[chip8->inst.X]] == false){
						chip8->PC += 2;
					}

					break;

				default:
					break;

			}
			break;

		case 0x0F:
			switch (chip8->inst.NN)
			{
			case 0x0A:
				// 0xFX0A
				// A key press is awaited, and then stored in VX (blocking operation, all instruction halted until next key event)

				bool key_pressed = false;

				for(uint8_t i = 0; i < sizeof(chip8->keypad); i++){
					if(chip8->keypad[i]){
						chip8->V[chip8->inst.X] = i;
						key_pressed = true;
						break;
					}
				}

				// keep getting the current opcode and running the this instruction until a key is pressed
				if(key_pressed == false){
					chip8->PC -= 2;
				}
				
				break;

			case 0x1E:
				// 0xFX1E
				// Adds VX to I. VF is not affected
				chip8->I += chip8->V[chip8->inst.X];
				break;
			
			case 0x07:
				// 0xFX07
				// Sets VX to the value of the delay timer

				chip8->V[chip8->inst.X] = chip8->delay_timer;

				break;

			case 0x15:
				// 0xFX15
				// Sets the delay timer to VX

				chip8->delay_timer = chip8->V[chip8->inst.X];

				break;

			case 0x18:
				// 0xFX18
				// Sets the sound timer to VX

				chip8->sound_timer = chip8->V[chip8->inst.X];

				break;

			case 0x29:
				// 0xFX29
				// Sets I to the location of the sprite for the character in VX. Characters 0-F (in hexadecimal) are represented by a 4x5 font

				chip8->I = chip8->V[chip8->inst.X] * 5;

				break;

			case 0x33:
				// 0xFX33
				uint8_t bcd = chip8->V[chip8->inst.X];

				chip8->ram[chip8->I + 2] = bcd % 10;
				bcd /= 10;

				chip8->ram[chip8->I + 1] = bcd % 10;
				bcd /= 10;

				chip8->ram[chip8->I] = bcd % 10;

				break;

			case 0x55:
				// 0xFX55
				// Stores from V0 to VX (including VX) in memory, starting at address I. The offset from I is increased by 1 for each value written, but I itself is left unmodified

				for(uint8_t i = 0; i <= chip8->inst.X; i++){
					chip8->ram[chip8->I+i] = chip8->V[i];
				}

				break;

			case 0x65:
				// 0xFX65
				// Fills from V0 to VX (including VX) with values from memory, starting at address I. The offset from I is increased by 1 for each value read, but I itself is left unmodified

				for(uint8_t i = 0; i <= chip8->inst.X; i++){
					chip8->V[i] = chip8->ram[chip8->I+i];
				}

				break;

			default:
				break;
			}

			break;

		default:
			break;
	}

}
```

## main() 
So now we have done everything, developed all the necessary function, emulated all the instructions and configured sound now we just have to call everything inside the main() function.

```c
int main(int argc, char **argv){
	// NO ROM PASSED
	if(argc<2){
		printf("NO ROM PASSED\n");
		exit(EXIT_FAILURE);
	}


	// Emulator Config
	config_t config = {0};
	if(set_config(&config, argc, argv) == false){
		exit(EXIT_FAILURE);
	}

	// Initialization
	sdl_t sdl = {0};
	if(init_sdl(&sdl, &config) == false){
		exit(EXIT_FAILURE);
	}

	// CHIP-8 Initialization
	chip8_t chip8 = {0};
	const char *rom_name = argv[1];
	if(!init_chip8(&chip8, rom_name)){
		exit(EXIT_FAILURE);
	}

	clear_screen(sdl, config);

	srand(time(NULL));

	// Main Emulator Loop
	while(chip8.state != QUIT){
		// User Input
		handle_input(&chip8);

		if(chip8.state == PAUSED){continue;}

		// Get time before running instructions
		uint64_t start = SDL_GetPerformanceCounter();

		for(uint32_t i = 0; i < config.inst_per_sec/60; i++)
		emulate_instructions(&chip8, config);

		// Get time after running instructions
		uint64_t end = SDL_GetPerformanceCounter();


		double time_elapsed = (double)((end-start)*1000)/SDL_GetPerformanceFrequency();

		// Delay for 60fps
		SDL_Delay(16.67f > time_elapsed ? 16.67f - time_elapsed : 0);
		// Update Window
		if(chip8.draw){
			update_screen(sdl, config, chip8);
			chip8.draw = false;
		}
		update_timers(sdl, &chip8);
	}

	final_cleanup(sdl);

	exit(EXIT_SUCCESS);
}
```

### Results

#### Test ROMS

```bash
./bin/chip8 ./roms/test_opcode.ch8
```

![test_opcode](/chip8/test1.png)

```bash
./bin/chip8 ./roms/BC_test.ch8
```

![BC_test](/chip8/test2.png)

```bash
./bin/chip8 ./roms/IBM\ Logo.ch8
```

![BC_test](/chip8/ibm.png)

#### Tetris

```bash
./bin/chip8 ./roms/Tetris\ \[Fran\ Dachille\,\ 1991\].ch8
```

![tetris](/chip8/tetris.png)

#### Breakout

```bash
./bin/chip8 ./roms/Brix\ \[Andreas\ Gustafsson\,\ 1990\].ch8
```

![breakout](/chip8/breakout.png)

#### TANK

```bash
./bin/chip8 ./roms/Tank.ch8
```

![tank](/chip8/tank.png)

## Some Other Useful Resources
- [Cowgod CHIP8 Technical Reference (must read)](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)
- [Wikipedia](https://en.wikipedia.org/wiki/CHIP-8)
- [My Implmentation Source Code](https://github.com/suryapsp/chip8/)
- [Building a CHIP8 Emulator C++](https://austinmorlan.com/posts/chip8_emulator/)
- [Guide to making a CHIP8 Emulator](https://tobiasvl.github.io/blog/write-a-chip-8-emulator/)
