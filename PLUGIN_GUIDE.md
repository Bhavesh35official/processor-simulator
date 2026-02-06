# Processor Simulator Platform - Plugin Architecture

## Overview
This platform is designed to be extensible. You can add your own processor designs as JavaScript modules that plug into the simulator.

## Processor Plugin Structure

Each processor is a JavaScript object with the following structure:

```javascript
const MY_PROCESSOR = {
  // Unique identifier
  id: 'my-processor-v1',
  
  // Display name
  name: 'My Custom Processor',
  
  // Brief description
  description: 'A 16-bit RISC processor with 8 registers',
  
  // Register configuration
  registers: ['R0', 'R1', 'R2', 'R3', 'R4', 'R5', 'R6', 'R7', 'PC', 'SP'],
  
  // Memory configuration
  memory: {
    size: 65536,      // Total memory size in bytes
    wordSize: 16      // Bits per word
  },
  
  // Compiler function: converts C/C++ code to assembly
  compile: (sourceCode) => {
    // Your compilation logic here
    // Return array of instruction objects
    return [
      {
        addr: '0x0000',           // Memory address (hex string)
        instruction: 'MOV R0, #5', // Human-readable assembly
        binary: '0001000000000101' // Binary representation
      },
      // ... more instructions
    ];
  },
  
  // Execution function: simulates instruction execution
  execute: (instruction, currentState) => {
    // instruction: the current instruction object
    // currentState: { registers: {...}, memory: [...], pc: 0 }
    
    // Your execution logic here
    // Parse the instruction, update state
    
    // Return new state
    return {
      registers: { /* updated registers */ },
      memory: [ /* updated memory */ ],
      pc: currentState.pc + 1
    };
  },
  
  // Optional: Initial state
  initialState: {
    registers: { R0: 0, R1: 0, /* ... */ },
    memory: Array(256).fill(0),
    pc: 0
  }
};
```

## Adding Your Processor

1. **Create your processor module** in a separate file (e.g., `my-processor.js`)
2. **Export the processor object**
3. **Import and add to the PROCESSORS array** in the main simulator:

```javascript
import MY_PROCESSOR from './my-processor.js';

const PROCESSORS = [
  DEMO_PROCESSOR,
  MY_PROCESSOR,  // Add your processor here
];
```

## Compiler Implementation Guide

The `compile` function is where you'll implement your C/C++ to assembly translation. Here are some approaches:

### Option 1: Simple Pattern Matching (Good for basic subset of C)
```javascript
compile: (sourceCode) => {
  const assembly = [];
  const lines = sourceCode.split('\n');
  
  lines.forEach(line => {
    // Match variable declarations: int x = 5;
    const varMatch = line.match(/int\s+(\w+)\s*=\s*(\d+)/);
    if (varMatch) {
      const [_, varName, value] = varMatch;
      assembly.push({
        addr: `0x${assembly.length.toString(16).padStart(4, '0')}`,
        instruction: `MOV ${varName}, #${value}`,
        binary: encodeInstruction('MOV', varName, value)
      });
    }
    
    // Match arithmetic: c = a + b;
    const addMatch = line.match(/(\w+)\s*=\s*(\w+)\s*\+\s*(\w+)/);
    if (addMatch) {
      const [_, dest, src1, src2] = addMatch;
      assembly.push({
        addr: `0x${assembly.length.toString(16).padStart(4, '0')}`,
        instruction: `ADD ${dest}, ${src1}, ${src2}`,
        binary: encodeInstruction('ADD', dest, src1, src2)
      });
    }
  });
  
  return assembly;
}
```

### Option 2: Abstract Syntax Tree (More robust)
```javascript
compile: (sourceCode) => {
  // 1. Tokenize
  const tokens = tokenize(sourceCode);
  
  // 2. Parse into AST
  const ast = parse(tokens);
  
  // 3. Generate assembly from AST
  const assembly = generateAssembly(ast);
  
  return assembly;
}
```

### Option 3: Use Existing Compiler (Advanced)
You could integrate with existing compiler tools like:
- Write a custom backend for LLVM
- Use WebAssembly as intermediate representation
- Adapt an existing educational compiler

## Simulator Implementation Guide

The `execute` function simulates how your processor executes instructions:

```javascript
execute: (instruction, state) => {
  const newState = { ...state };
  
  // Parse instruction
  const parts = instruction.instruction.split(/[\s,]+/);
  const opcode = parts[0];
  
  switch(opcode) {
    case 'MOV':
      const dest = parts[1];
      const value = parseInt(parts[2].replace('#', ''));
      newState.registers[dest] = value;
      break;
      
    case 'ADD':
      const rd = parts[1];
      const rs1 = parts[2];
      const rs2 = parts[3];
      newState.registers[rd] = 
        newState.registers[rs1] + newState.registers[rs2];
      break;
      
    case 'LOAD':
      const reg = parts[1];
      const addr = parseInt(parts[2]);
      newState.registers[reg] = newState.memory[addr];
      break;
      
    case 'STORE':
      const sreg = parts[1];
      const saddr = parseInt(parts[2]);
      newState.memory[saddr] = newState.registers[sreg];
      break;
      
    // ... more instruction implementations
  }
  
  // Increment program counter
  newState.pc = (state.pc || 0) + 1;
  
  return newState;
}
```

## Example: Simple 4-bit Processor

Here's a complete example of a minimal processor:

```javascript
const SIMPLE_4BIT = {
  id: 'simple-4bit',
  name: 'Simple 4-bit Processor',
  description: 'Educational 4-bit processor with 4 registers and 8 instructions',
  
  registers: ['R0', 'R1', 'R2', 'R3'],
  
  memory: {
    size: 256,
    wordSize: 8
  },
  
  // Instruction set:
  // 0000 RRRR VVVV VVVV - LOAD Rx, #immediate
  // 0001 RRDD 0000 0000 - ADD Rd, Rx
  // 0010 RRDD 0000 0000 - SUB Rd, Rx
  // 1111 0000 0000 0000 - HALT
  
  compile: (code) => {
    // Very simple: just recognize patterns
    const assembly = [];
    
    // Match: int x = NUMBER;
    const assignments = code.matchAll(/int\s+(\w+)\s*=\s*(\d+)/g);
    for (const [_, varName, value] of assignments) {
      const regNum = varName.charCodeAt(0) - 'a'.charCodeAt(0);
      assembly.push({
        addr: `0x${assembly.length.toString(16).padStart(2, '0')}`,
        instruction: `LOAD R${regNum}, #${value}`,
        binary: `0000${regNum.toString(2).padStart(4, '0')}${parseInt(value).toString(2).padStart(8, '0')}`
      });
    }
    
    assembly.push({
      addr: `0x${assembly.length.toString(16).padStart(2, '0')}`,
      instruction: 'HALT',
      binary: '1111000000000000'
    });
    
    return assembly;
  },
  
  execute: (instruction, state) => {
    const binary = instruction.binary;
    const opcode = binary.substring(0, 4);
    const newState = JSON.parse(JSON.stringify(state));
    
    if (opcode === '0000') { // LOAD
      const reg = parseInt(binary.substring(4, 8), 2);
      const value = parseInt(binary.substring(8, 16), 2);
      newState.registers[`R${reg}`] = value;
    }
    
    return newState;
  }
};
```

## Testing Your Processor

Before integrating, test your processor module:

```javascript
// Test compilation
const testCode = 'int a = 5; int b = 3; int c = a + b;';
const assembly = MY_PROCESSOR.compile(testCode);
console.log('Assembly:', assembly);

// Test execution
let state = { registers: { R0: 0, R1: 0 }, memory: [] };
assembly.forEach(instruction => {
  state = MY_PROCESSOR.execute(instruction, state);
  console.log('State:', state);
});
```

## Next Steps

1. **Start simple**: Implement basic MOV, ADD, SUB instructions first
2. **Test incrementally**: Add one instruction type at a time
3. **Expand gradually**: Add jumps, branches, memory operations
4. **Optimize**: Once working, improve your compiler/simulator

## Resources for Building Processors

- **Instruction Set Architecture (ISA) Design**:
  - MIPS: Simple RISC architecture (good reference)
  - ARM: Modern efficient design
  - RISC-V: Open source, modular
  
- **Compiler Theory**:
  - Dragon Book (Compilers: Principles, Techniques, and Tools)
  - Crafting Interpreters (free online)
  
- **Simulation**:
  - Study existing simulators (MARS for MIPS, Venus for RISC-V)
  - Consider cycle-accurate vs functional simulation

Good luck building your processor! 🚀
