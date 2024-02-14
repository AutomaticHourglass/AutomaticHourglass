---
layout: post
title: A Centralized Queue System with Terminal and Python
subtitle: Keep your production environment busy from your cellphone
tags:
  - queue
  - bash
  - python
  - gsheet
  - production
comments: true
author: Dr. Ünsal Gökdağ
---
## TL;DR
You can run multiple resources (GPU) consuming a central queue system(gsheet) with minimal intervention (mobile phone) 24/7 any place, any time (again, mobile phone) and also, check its status (logging, wandb).

Interested in the solution? [Look at the end of the article](## PS: Whole Code)

Interested in the process? Read along..


## The Problem
I'm currently doing *a lot* of experiments on [runpod](https://runpod.io?ref=7mip03at) and I have realised that I'm constantly using at least 2 GPUs, all the time. I remember getting up from sleep and making typing a simple run script, changing a few characters and the going back to sleep and I did not liked this experience.
So I started dabbling with chatGPT to create a simple bash script that reads a file which is literally called `gpu_queue.txt`, reads the first line and execute it. After a few minutes, a running version was working as intended.
What I have started as a requirement is that multiple version of this should be able to run concurrently without firing up lines from the queue unless the gpu is free of use. They way checking that required me to use locks which I liked learning from chatGPT but the end solution became something *different* but *better* a bit in my opinion.


## The Solution
I have thought about the ideal solution should have these:
- Ability to be used by multiple consumers (GPUs, bash terminals, etc)
- Utilise the GPUs in a good extent (more on that later)
- It should run as long as the queue feeds instructions to to it.
- I should control be to control it 24/7 which is via my cellphone

In the end, I have managed to achieve all of these requirements in less than 100 lines of code in 2 files: `runner.sh` and `read_queue.py`.

The bash script consists of multiple parts:
``` bash
while true; do
    gpu=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits)
    if [ "$gpu" -lt 50 ]; then
        echo "GPU utilization is below 50%. Executing queue..."
        execute_queue
    else
        echo "GPU utilization is above 50%. Waiting..."
    fi
    sleep 5  # Add a small sleep to reduce flicker
done
```
This part is a while loop with a 5 second delay that executes the queue if the gpu is not utilised (<50%).

Execute Queue consists of 4 parts:
- Reading an element from the queue
- Executing this with first additional parameters: To estimate batch size
	- Measuring maximum gpu memory during this period
- Executing this with second additional parameters: To make the actual run

Lets go over the this party by part:
### Reading an Element from the Queue

This part is up to your taste and/or requirements.

I have used the way satisfies all of my requirements. I'm reading the queue from a gsheet that I own. The ghseet is quite simple, it consists of one line of text entries as being the "Queue" and one *counter* which denotes the current location in the queue.

![[gsheet_screenshot.png]]
![]({{ 'assets/2024-02-14-A-Centralized-Queue-System-with-Terminal-and-Python/gsheet_screenshot.png' | relative_url }})

The reader program is as follows:
``` python
import gspread

def read_increment_and_update(sheet_key):
    # Open the Google Sheet by key without using credentials
    gc = gspread.service_account("your-secret-file.json")
    sheet = gc.open_by_key(sheet_key)

    # Read the current counter value from cell B1
    counter_cell = sheet.sheet1.acell("B1")
    current_counter = int(counter_cell.value) if counter_cell.value else 1

    # Read the corresponding row based on the counter
    row_to_read = current_counter + 2
    data_cell = sheet.sheet1.acell(f"C{row_to_read}")
    first_row = data_cell.value if data_cell.value else ""

    # Increment the counter and update cell B1
    new_counter = current_counter + 1
    counter_cell.value = str(new_counter)
    sheet.sheet1.update_cells([counter_cell])

    # Update the corresponding row with the read data
    data_cell.value = first_row
    sheet.sheet1.update_cells([data_cell])

    return first_row


if __name__ == "__main__":
    # Replace 'YOUR_SHEET_KEY' with the actual key of your public Google Sheet
    result = read_increment_and_update("gsheet-id")
    print(result)

```

Quite simple, just reading `B1` cell and offsetting the line to read from `C` column. You can make it as complicated as you want of course.

![[share_with_service_account.png]]
![]('assets/2024-02-14-A-Centralized-Queue-System-with-Terminal-and-Python/share_with_service_account.png' | relative_url )
*don't forget to add your service account as writer to the gsheet*

This was the only python part needed, to read and write to gsheet. Feel free to make a pure bash implementation if you'd like (but make it simple please, otherwise, we know everything is possible..).

## Executing it with first additional parameters
Since my current runs are nanogpt executions, I can give any global variable from command line as an override to the system and it will show like this:
![[gpt_params.png]]
![]({{'https://automatichourglass.github.io/AutomaticHourglass/assets/2024-02-14-A-Centralized-Queue-System-with-Terminal-and-Python/gpt_params.png' | relative_url}})
*production parameters, logging all my debug runs?(no I'm not, wandb)*

But estimating batch_size from get go is not that easy and I wanted to challenge that. What if I make a run for 1-2 minutes with a predefined `batch_size`, then measure the gpu memory and estimate the maximum possible `batch_size` with a safety margin? That would eliminate the estimation of the `batch_size` and also allows us to constantly run `k` runs by dividing the GPU RAM among `k` pieces. Since my experiments are process bound instead of memory bound, I tend to utilise the whole gpu and make `k` equal to `1`.

Another problem comes with this is the nanogpt runs are a single process that can't be interrupted. The solution to this problem is two fold:
- fire up a subprocess that measures the GPU usage
- wait for response to catch up with the original process

So my main process will fire up an experiment with *additional* parameters that *overrides* the *overridden* parameters (logging, batch size) but the remaining parameters would be the same so that we can reliably measure the memory consumption. Since the gpu usage is 0 on process beginning and end, how would you measure the maximum GPU RAM usage during this short training period?
``` bash
# Override arguments for the first attempt
OVERRIDES=" --max_tokens=1000000 --wandb_log=False"
```

### Making Subprocesses
By making subprocesses! The reason is this;
At any given point in time, you can measure the current gpu usage by probing `nvidia-smi` function in a subprocess but at which point? But what if we take 12 samples one after another every 10 seconds for a total of 2 minutes and return the maximum value we see during this time period?

``` bash
max_gpu_usage() {
  max_usage=0
  for ((i = 0; i < 12; i++)); do
    current_usage=$(nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits)
    ((current_usage > max_usage)) && max_usage=$current_usage
    sleep 10
  done
  echo $max_usage
}
```


Funny how processes pass arguments between each other in bash, its pure `stdout`, literally what you print to the screen can be passed as a parameter to the next call in the event loop like this:
``` bash
gpu_memory_usage=$(max_gpu_usage)
```
and then use it as a variable in the bash script.

### Estimating Maximum Batch Size
If the run is successful, it should end around 1-2 minutes and during this time your GPU usage probe should be able to get the maximum GPU usage during *both* your training and validation cycles to have a healthy estimation of the current RAM usage of the experiment.

``` bash
# Function to calculate batch size based on GPU memory
calculate_batch_size() {
    local gpu_memory_usage=$1

    # Maximum available GPU memory retrieved from nvidia-smi
    local max_gpu_memory=$(nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits)

    # Default batch size, this number should be same with the one in the config file by default
    local default_batch_size=8

    # Calculate the maximum batch size that fits within available GPU memory
    local max_batch_size=$((max_gpu_memory * default_batch_size * 9 / 10 / gpu_memory_usage))

    # Round down to the nearest multiple of default_batch_size
    local rounded_batch_size=$((max_batch_size / default_batch_size * default_batch_size))

    # Ensure the calculated batch size is within reasonable bounds
    if [ "$rounded_batch_size" -gt 0 ] && [ "$rounded_batch_size" -le 4096 ]; then
        echo "$rounded_batch_size"
    else
        echo "$default_batch_size"
    fi
}
```

I have found out that adding an additional constant like `0.9` or `*9/10` to the calculation gave better overall performance since there were no failing runs due to GPU RAM size after that.

I vaguely remember that having `batch_size` as a multiple of `16` gave much better throughput so I added this line in the code:
``` bash
# Round down to the nearest multiple of default_batch_size
    local rounded_batch_size=$((max_batch_size / default_batch_size * default_batch_size))
```


By defining a function to do basic arithmetic, we gather the gpu usage and put it inside the function:
``` bash
# Calculate batch size based on GPU memory
batch_size=$(calculate_batch_size "$gpu_memory_usage")

# Wait for the override command to finish
wait "$pid"
```
we estimate a raliable maximum batch size for this experiment.

## Making the Actual Run
I have quickly realised that this will create checkpoints of multiple models and they all will have name `out.ckpt` which will get overwritten by each subsequent run. To eliminate that, I need to create a unique identifier during the run but I have found out that making this over the bash is super simple so I have decided to make it that way:
``` bash
# Generate a new random 8-digit hexadecimal string for the 'id' variable
id=$(openssl rand -hex 4)
```

So I pass the `id` variable to the program which saves the output checkpoint with that suffix. The whole code chunk looks as follows:
``` bash
# Execute the original command without overrides, including the dynamically calculated batch size
if eval "$command --batch_size=$batch_size --version=$id"; then
	echo "Command successfully executed."
	echo "$command" >> "$FINISHED_FILE"
else
	echo "Command execution without overrides failed."
fi
```

## Closing Remarks
This was actually fun and very teaching for me as now I have a responsibility to keep *the queue* full at all times and plan for the future and estimate how much free time I have before adding new jobs to the *queue*.

As usual, I thank chatGPT for making my life very easy throughout the process but keeping me vigilant by frequent hallucinations or forgetting variables that existed or discussed about.

---
## PS: Whole Code
file: `read_queue.py`
``` python
import gspread

def read_increment_and_update(sheet_key):
    # Open the Google Sheet by key without using credentials
    gc = gspread.service_account("your-secret-file.json")
    sheet = gc.open_by_key(sheet_key)

    # Read the current counter value from cell B1
    counter_cell = sheet.sheet1.acell("B1")
    current_counter = int(counter_cell.value) if counter_cell.value else 1

    # Read the corresponding row based on the counter
    row_to_read = current_counter + 2
    data_cell = sheet.sheet1.acell(f"C{row_to_read}")
    first_row = data_cell.value if data_cell.value else ""

    # Increment the counter and update cell B1
    new_counter = current_counter + 1
    counter_cell.value = str(new_counter)
    sheet.sheet1.update_cells([counter_cell])

    # Update the corresponding row with the read data
    data_cell.value = first_row
    sheet.sheet1.update_cells([data_cell])

    return first_row


if __name__ == "__main__":
    # Replace 'YOUR_SHEET_KEY' with the actual key of your public Google Sheet
    result = read_increment_and_update("gsheet-id")
    print(result)
```

file: `runner.sh`
``` bash
#!/bin/bash


max_gpu_usage() {
  max_usage=0
  for ((i = 0; i < 12; i++)); do
    current_usage=$(nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits)
    ((current_usage > max_usage)) && max_usage=$current_usage
    sleep 10
  done
  echo $max_usage
}

# File path for finished commands
FINISHED_FILE="finished.txt"

# Function to calculate batch size based on GPU memory
calculate_batch_size() {
    local gpu_memory_usage=$1

    # Maximum available GPU memory retrieved from nvidia-smi
    local max_gpu_memory=$(nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits)

    # Default batch size
    local default_batch_size=8

    # Calculate the maximum batch size that fits within available GPU memory
    local max_batch_size=$((max_gpu_memory * default_batch_size * 9 / 10 / gpu_memory_usage))

    # Round down to the nearest multiple of default_batch_size
    local rounded_batch_size=$((max_batch_size / default_batch_size * default_batch_size))

    # Ensure the calculated batch size is within reasonable bounds
    if [ "$rounded_batch_size" -gt 0 ] && [ "$rounded_batch_size" -le 4096 ]; then
        echo "$rounded_batch_size"
    else
        echo "$default_batch_size"
    fi
}

# Function to execute commands from the GPU queue and append to finished.txt
execute_queue() {
    echo "Executing GPU queue..."
    while IFS= read -r command; do
        echo "Executing command: $command"

        # Generate a new random 8-digit hexadecimal string for the 'id' variable
        id=$(openssl rand -hex 4)

        # Run the override command in the background
        eval "$command $OVERRIDES" &
        pid=$!  # Capture the process ID

        # Get GPU memory usage using nvidia-smi as a subprocess during the override
        gpu_memory_usage=$(max_gpu_usage)

        # Calculate batch size based on GPU memory
        batch_size=$(calculate_batch_size "$gpu_memory_usage")

        # Wait for the override command to finish
        wait "$pid"

        # Execute the original command without overrides, including the dynamically calculated batch size
        if eval "$command --batch_size=$batch_size --version=$id"; then
            echo "Command successfully executed."
            echo "$command" >> "$FINISHED_FILE"
        else
            echo "Command execution without overrides failed."
        fi
    done < <(python read_queue.py)
    echo "GPU queue executed."
}

# Override arguments for the first attempt
OVERRIDES=" --max_tokens=1000000 --wandb_log=False"

while true; do
    nvidia_smi_output=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits)
    if [ "$nvidia_smi_output" -lt 50 ]; then
        echo "GPU utilization is below 50%. Executing queue..."
        execute_queue
    else
        echo "GPU utilization is above 50%. Waiting..."
    fi
    sleep 5  # Add a small sleep to reduce flicker
done

```
