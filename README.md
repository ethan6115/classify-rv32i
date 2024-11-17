# Assignment 2: Classify

## abs 
I use a bqe to test whether the number is greater than zero. If the number smaller than zero, I use (0 - a) = -a to get the positive.  
```c
abs:
    # Prologue
    ebreak
    # Load number from memory
    lw t0 0(a0)
    bge t0, zero, done
    sub t0, zero, t0	#if negative, use(0-t0) to get positive
    sw t0, 0(a0)

done:
    # Epilogue
    jr ra
```
## argmax 
I use a loop to iterate through the entire array and utilize two registers to store the maximum value and its position. If I encounter a larger value during the loop, I update these two registers and keep iterate until the entire array is processed.  
```c
loop_start:
    bge t2, a1, end_loop       # if t2 >= a1, end loop
    slli t3, t2, 2	       #load a0+t2*4
    add t3, t3, a0
    lw t4, 0(t3)
    bge t0, t4, smaller		#if t0>=t4, jump to smaller
    addi t0, t4, 0		#update t0 t1
    addi t1, t2, 0
smaller:
    addi t2, t2, 1
    j loop_start
end_loop:
    addi a0, t1, 0
    jr ra
```
## relu
I use a loop to iterate through the entire array and check whether the number is greater than zero. If the current value is less than 0, store 0 at the current array address to overwrite the original value. 
```c
loop_start:
    bge t1, a1, end_loop	#if t1 >= a1, end loop
    slli t2, t1, 2	        #load a0+t1*4
    add t2, t2, a0
    lw t3, 0(t2)
    bge t3, x0, positive	#jump to positive if t3>=0
    sw x0, 0(t2)		#update element to 0
positive:
    addi t1, t1, 1
    j loop_start
end_loop:
    jr ra
```
## dot
I use two registers, s0 and s1, to store the strides (i * stride). This eliminates the need to calculate the stride within each loop iteration.

For multiplication, I implemented it manually. First, I record the sign bits of the multiplier and multiplicand. Then, I convert both values to their absolute values and perform the multiplication manually. The manual multiplication is achieved using a loop that performs repeated addition until the accumulator matches the multiplier. Finally, I use an XOR operation on the two sign bits to determine the final sign of the result.
```c
    addi sp, sp, -8           #store s0 and s1 in stack  
    sw s0, 0(sp)                 
    sw s1, 4(sp)  
    li s0, 0
    li s1, 0

loop_start:
    bge t1, a2, loop_end

#get arr0[i * stride0] = arr0[t1 * a3]
    slli t2, s0, 2                 # t2 = (i * stride0) * 4
    add t2, a0, t2                  
    lw t3, 0(t2)                   # t3 = arr0[i * stride0]
    
# get arr1[i * stride1] = arr1[t1 * a4]
    slli t2, s1, 2                  
    add t2, a1, t2                  
    lw t4, 0(t2)                    

#do (arr0[i * stride0] * arr1[i * stride1])
# see if  t3 negative
    srai t2, t3, 31           # get the sign
    xor t3, t3, t2            # get the absolute value
    sub t3, t3, t2            

# see if  t4 negative
    srai t5, t4, 31           # get the sign
    xor t4, t4, t5            # get the absolute value
    sub t4, t4, t5            

    xor t6, t2, t5            # see if multiplier and multiplicand have different sign
    li t2, 0                   
    li t5, 0
multiply_loop:
    bge t5, t3, multiply_done  
    add t2, t2, t4            # t2 += t4
    addi t5, t5, 1            # t5++
    j multiply_loop            

multiply_done:
    bnez t6, negate_result    # jump if not different 
    j end_multiply

negate_result:
    sub t2, x0, t2            # t2 = 0 - t2

end_multiply: 
    add t0, t0, t2                 
    addi t1, t1, 1            # i++
    add s0, s0, a3
    add s1, s1, a4
    j loop_start
    
loop_end:
    lw s0, 0(sp)                #load s0 and s1 from stack
    lw s1, 4(sp)                
    addi sp, sp, 8              
    
    addi a0, t0, 0
    jr ra
```
## matmul
There are only a few modifications needed in matmul.s, specifically at the end of the inner loop and the outer loop. At the end of the inner loop, I increment the outer loop counter and move the pointer for matrix A to the next row, then jump back to outer_loop_start. At the end of the outer loop, I restore the saved register values from the stack and return.
```c
inner_loop_end:
    addi s0, s0, 1
    slli t2, a2, 2
    add s3, s3, t2                 # update matrix A pointer
    j outer_loop_start
    
outer_loop_end:
    
    # end the program
    lw ra, 0(sp)
    lw s0, 4(sp)
    lw s1, 8(sp)
    lw s2, 12(sp)
    lw s3, 16(sp)
    lw s4, 20(sp)
    lw s5, 24(sp)
    addi sp, sp, 28
    jr ra
```
## read/write matrix
In read_matrix.s and write_matrix.s, I simply used the manual multiplication to replace the mul instruction.
## classify
In classify.s, I wrote a mul function like below to implement manual multiplication. This was the original manual multiplication I used in dot.s, read_matrix.s and write_matrix.s.
```c
mul:
    li t0, 0                   
    li t1, 0                            

multiply_loop:
    bge t1, a1, multiply_done # 
    add t0, t0, a0            # t0 += a0
    addi t1, t1, 1            # t1++
    j multiply_loop            

multiply_done:
    mv a0, t0
    jr ra
```
While using the mul function above, I couldn't pass the test_classify_3_print test. Then I realized I had forgotten to account for negative numbers, as the mul function above only handles positive multiplication. To fix this error, I wrote a second version to replace the manual multiplication in dot.s, read_matrix.s, and write_matrix.s.
```c
mul:
    li t0, 0                   
    li t1, 0                  

    # see if  a1 negative
    srai t2, a1, 31           # get the sign
    xor a1, a1, t2            # get the absolute value
    sub a1, a1, t2            

    # see if  a0 negative
    srai t3, a0, 31           # get the sign
    xor a0, a0, t3            # get the absolute value
    sub a0, a0, t3            

multiply_loop:
    bge t1, a1, multiply_done # 
    add t0, t0, a0            # t0 += a0
    addi t1, t1, 1            # t1++
    j multiply_loop            

multiply_done:
    xor t4, t2, t3            # see if multiplier and multiplicand have different sign
    bnez t4, negate_result    # jump if not different 
    j end_multiply

negate_result:
    sub t0, x0, t0            # t0 = 0 - t0

end_multiply:
    mv a0, t0                  
    jr ra                     
```
