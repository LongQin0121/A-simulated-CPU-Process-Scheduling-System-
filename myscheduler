#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <stdbool.h>

// ------------------------ Constants for maximum values ------------------------

#define MAX_DEVICES 4
#define MAX_DEVICE_NAME 20
#define MAX_COMMANDS 10
#define MAX_COMMAND_NAME 20
#define MAX_SYSCALLS_PER_PROCESS 40
#define MAX_RUNNING_PROCESSES 50
#define DEFAULT_TIME_QUANTUM 100
#define TIME_CONTEXT_SWITCH 5
#define TIME_CORE_STATE_TRANSITIONS 10      // Time taken for core state transitions
#define TIME_ACQUIRE_BUS 20                 // Time taken to acquire the bus 
#define CHAR_COMMENT '#'

//------------------------ Self-defined structures  ------------------------

// Structures for representing devices , syscalls  and commands 
enum syscall_type
{
    READ,
    WRITE,
    SLEEP,
    WAIT,
    EXIT,
    SPAWN,
};

enum process_state
{
    New,
    Ready,
    Blocked,
    Running,
    Exit,
};

typedef struct Device
{
    char device_name[MAX_DEVICE_NAME];
    int  readspeed;
    int  writespeed;
} Device;

typedef struct Syscall
{
    int run_time;                                      // represents the runtime (execution time) of a specific syscall
    enum syscall_type type;                            // The type of the syscall (e.g., READ, WRITE, SLEEP)
    char syscall_target[MAX_DEVICE_NAME];
    char spawn_command[MAX_COMMAND_NAME];              // The command to spawn
    int io_size;                                       // The size of I/O operation (if applicable)
    int sleep_time;
} Syscall;

typedef struct Command
{
    char command_name[MAX_COMMAND_NAME];                     // The name of the command
    int num_syscalls;                                   
    Syscall syscalls[MAX_SYSCALLS_PER_PROCESS];
    enum process_state state;
    int elapsed_cpu_time;                                    // track the total CPU time consumed by a process.
    int executed_syscalls;                          

    // Fields for handling sleep syscall 
    int start_sleep_time;                                    // The time at which the command started sleeping.
    int sleep_duration; 

    //  Fields for handling spawn and wait syscalls
    int child_index[MAX_RUNNING_PROCESSES];                   // Index of child processes spawned by this command
    int num_children;                       
   
    // Fields for handling bus operations
    int  bus_size;
    int  bus_wait_start;                                      // Timestamp when the command started waiting for the bus
    int  bus_speed;
} Command;

// Structure for representing a queue of commands
typedef struct Queue
{
    int front;
    int rear;
    int size;
    int capacity;
    Command *commands[MAX_RUNNING_PROCESSES];
}Queue;

//------------------------ Global variables ------------------------

int time_quantum = DEFAULT_TIME_QUANTUM;
int num_commands = -1;
int total_SYS_time = 0;
int total_onCPU_time;
int last_bus_end;                            // The timestamp of the last completed bus operation
int num_devices = 0;

Device devices[MAX_DEVICES];
Command commands[MAX_COMMANDS];              // An array to store information about commands or processes 
Command *last_bus_command;                  // A pointer to the last command that used the system bus
Queue ready_queue;
Queue bus_queue;
Queue sleep_queue;
Queue wait_queue;


// ------------------------   Function prototypes    ------------------------ 
void read_sysconfig(char argv0[], char filename[]);
void read_commands(char argv0[], char filename[]);
void init_queues(void);
void execute_commands(void);
bool process_terminated(void);



// ------------------------   Main function    ------------------------ 
int main(int argc, char *argv[])
{
    //  ENSURE THAT WE HAVE THE CORRECT NUMBER OF COMMAND-LINE ARGUMENTS
    if (argc != 3)
    {
        printf("Usage: %s sysconfig-file command-file\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    //  READ THE SYSTEM CONFIGURATION FILE
    read_sysconfig(argv[0], argv[1]);

    //  READ THE COMMAND FILE
    read_commands(argv[0], argv[2]);

    // Initialize all the queues
    init_queues();

    // Execute the commands
    execute_commands();

    printf("measurements  %d  %d\n", total_SYS_time, total_onCPU_time * 100 / total_SYS_time);
    exit(EXIT_SUCCESS);
}


//  ------------------  Functions:  read sysconfig and commands files -----------------------------

void read_sysconfig(char argv0[], char filename[])
{
    char line[300];   
    FILE* sysconfig=fopen(filename,"r");
    if (sysconfig == NULL)
    {
        printf("Error opening file %s\n", filename);
        exit(EXIT_FAILURE);
    }
    // use fgets() to process line by line
    while (fgets(line, sizeof(line), sysconfig))
    {
        // if line is a comment, skip it
        if (line[0] == CHAR_COMMENT)
        {
            continue;
        }
        // if line is a device, add it to the devices array
        if (line[0] == 'd')
        {
            sscanf(line, "device %s %dBps %dBps", devices[num_devices].device_name, &devices[num_devices].readspeed, &devices[num_devices].writespeed);
            num_devices++;
        }
        // if line is a timequantum, renew the time quantum
        if (line[0] == 't')
        {
            sscanf(line, "timequantum %dusec", &time_quantum);
        }
    }
}

//  ------------ read and parse the file, populate the commands array with information about commands and their associated system calls  --------------------

void read_commands(char argv0[], char filename[])
{   
    // open and check the file
    char line[300];   
    FILE* fp_comm=fopen(filename,"r");
    if (fp_comm == NULL)
    {
        printf("Failed opening file %s\n", filename);
        exit(EXIT_FAILURE);
    }

    // process the file line by line
    while (fgets(line, sizeof(line), fp_comm))
    {
        // if line is a comment, skip it
        if (line[0] == CHAR_COMMENT)
        {
            continue;
        }
        // if line is a command, add it to the commands array
        if (line[0] == '\t')
        {
            // it is a subsyscall,first get the syscall execution time and type
            int syscall_execution_time;
            char syscall_type_string[10];
            enum syscall_type syscall_type;
            sscanf(line, "\t%dusecs %s", &syscall_execution_time, syscall_type_string);
            // then get the syscall target and size
            if (strcmp(syscall_type_string, "read") == 0)
            {
                syscall_type = READ;
            }
            else if (strcmp(syscall_type_string, "write") == 0)
            {
                syscall_type = WRITE;
            }
            else if (strcmp(syscall_type_string, "sleep") == 0)
            {
                syscall_type = SLEEP;
            }
            else if (strcmp(syscall_type_string, "wait") == 0)
            {
                syscall_type = WAIT;
            }
            else if (strcmp(syscall_type_string, "exit") == 0)
            {
                syscall_type = EXIT;
            }
            else if (strcmp(syscall_type_string, "spawn") == 0)
            {
                syscall_type = SPAWN;
            }
            else
            {
                printf("Error: unknown syscall %s\n", syscall_type_string);
                exit(EXIT_FAILURE);
            }

            char syscall_target[MAX_DEVICE_NAME];
            char syscall_spawn_command[MAX_COMMAND_NAME];
            int syscall_size = 0;
            int sleep_time = 0;
           
            // if it is a read or write syscall, we also get the target and size
            if (syscall_type == READ || syscall_type == WRITE)
            {
                sscanf(line, "\t%dusecs %s %s %dB", &syscall_execution_time, syscall_type_string, syscall_target, &syscall_size);
            }
            // if it is a sleep syscall, we only get the sleep time
            else if (syscall_type == SLEEP)
            {
                sscanf(line, "\t%dusecs %s %dusecs", &syscall_execution_time, syscall_type_string, &sleep_time);
            }
            // if it is a spawn syscall, we only get the spawn command
            else if (syscall_type == SPAWN)
            {
                sscanf(line, "\t%dusecs %s %s", &syscall_execution_time, syscall_type_string, syscall_spawn_command);
            }
            // then add the syscall to the current command
            commands[num_commands].syscalls[commands[num_commands].num_syscalls].run_time = syscall_execution_time;
            commands[num_commands].syscalls[commands[num_commands].num_syscalls].type = syscall_type;
            strcpy(commands[num_commands].syscalls[commands[num_commands].num_syscalls].syscall_target, syscall_target);
            strcpy(commands[num_commands].syscalls[commands[num_commands].num_syscalls].spawn_command, syscall_spawn_command);
            commands[num_commands].syscalls[commands[num_commands].num_syscalls].io_size = syscall_size;
            commands[num_commands].syscalls[commands[num_commands].num_syscalls].sleep_time = sleep_time;
            commands[num_commands].num_syscalls++;
        }
        else if (strlen(line) > 1)     // it is a command
        {   
            num_commands++;
            sscanf(line, "%s", commands[num_commands].command_name);
        }
    }
}

//----------------------  Functions associated with Queue ---------------------

// Function to initialize a queue
void init_queue(Queue *q, int capacity)
{
    q->capacity = capacity;
    q->front = q->size = 0;
    q->rear = capacity - 1;
    // Initialize or reset the memory buffer q->commands by setting all its bytes to 0
    memset(q->commands, 0, sizeof(q->commands));
}

//initialize all four queues
void init_queues(void)
{
    init_queue(&ready_queue, MAX_RUNNING_PROCESSES);
    init_queue(&bus_queue, MAX_RUNNING_PROCESSES);
    init_queue(&sleep_queue, MAX_RUNNING_PROCESSES);
    init_queue(&wait_queue, MAX_RUNNING_PROCESSES);
}

// Function to check if a queue is full
int isFull(Queue *q)
{
    return (q->size == q->capacity);
}

// Function to check if a queue is empty
int isEmpty(Queue *q)
{
    return (q->size == 0);
}

// Function to enqueue a process in a queue
void enqueue(Queue *q,  Command *proc)
{
    if (isFull(q))
        return;
    q->rear = (q->rear + 1) % q->capacity;
    q->commands[q->rear] = proc;
    q->size ++;
}

//Function to pop a process out of a queue
Command *dequeue(Queue *q)
{
    if (isEmpty(q))
        return NULL;
    Command *proc = q->commands[q->front];
    q->commands[q->front] = NULL;
    q->front = (q->front + 1) % q->capacity;
    q->size--;
    return proc;
}

//------------------- Function to modify the state of process, handle the sleep,wait and bus Queue --------------------
/**
 * @brief change the state of a command, and update the system run time
 * @param command the process that is being modified
 * @param state the new state of the command
 */
void update_state( Command *command, enum process_state state)
{
    if (command->state == Running)
    {
        if (state == Ready || state == Blocked)
        {
           total_SYS_time += TIME_CORE_STATE_TRANSITIONS;
        }
    }
    if (command->state == Ready)
    {
        if (state == Running)
        {
           total_SYS_time += TIME_CONTEXT_SWITCH;
        }
    }
    if (command->state == Blocked)
    {
        if (state == Ready)
        {
           total_SYS_time += TIME_CORE_STATE_TRANSITIONS;
        }
    }
    if (state == Ready)
    {
        enqueue(&ready_queue, command);
    }
    command->state = state;
}

/**
 * @brief deal with  all the processes in the sleep queue, when a process wakes up, move the process to readyQueue
 *
 */
void handle_sleep_queue()
{
    Queue temp_queue;
    init_queue(&temp_queue, MAX_RUNNING_PROCESSES);
    // iterate through the sleep queue, find the commands that are ready to wake up
    while (!isEmpty(&sleep_queue))
    {
         Command *command = dequeue(&sleep_queue);
        if (total_SYS_time - command->start_sleep_time >= command->sleep_duration)
        {
            update_state(command, Ready);
        }
        else
        {
            enqueue(&temp_queue, command);
        }
    }
    // we reinqueue the commands that are not ready to the sleep queue
    while (!isEmpty(&temp_queue))
    {
         Command *command = dequeue(&temp_queue);
        enqueue(&sleep_queue, command);
    }
}

/**
 * @brief handle all the processes in the wait queue, if all its children have exited, move the process to readyQueue
 *
 */
void handle_wait_queue()
{
    Queue temp_queue;
    init_queue(&temp_queue, MAX_RUNNING_PROCESSES);
    // iterate through the wait queue, find the commands that all its children have exited
    while (!isEmpty(&wait_queue))
    {
         Command *command = dequeue(&wait_queue);
        int all_exited = 1;
        for (int i = 0; i < command->num_children; i++)
        {
            if (commands[command->child_index[i]].state != Exit)
            {
                all_exited = 0;
                break;
            }
        }
        // all its children have exited
        if (all_exited)
        {
            update_state(command, Ready);
        }
        else
        {
            enqueue(&temp_queue, command);
        }
    }
    // put other commands back to the wait queue
    while (!isEmpty(&temp_queue))
    {
         Command *command = dequeue(&temp_queue);
        enqueue(&wait_queue, command);
    }
}

/**
 * @brief process with the highest priority (determined by bus speed) is granted access to the shared bus, 
    and  it handles the timing and state transitions associated with bus access. 
    Other processes are temporarily removed from the queue and requeued if they do not have the highest priority for bus access.
 */

void handle_bus_queue()
{
    // if last bus command should finish
    if (last_bus_command != NULL && total_SYS_time >= last_bus_end)
    {
        update_state(last_bus_command, Ready);
        last_bus_command = NULL;
    }
    // some commands are still holding the bus, or no command is waiting for the bus
    if (last_bus_command != NULL || isEmpty(&bus_queue))
    {
        return;
    }
    Queue temp_queue;
    init_queue(&temp_queue, MAX_RUNNING_PROCESSES);
    // iterate through the bus queue, find the commands that can acquire the bus
    // it should be the one with highest speed and waiting for the longest time
    int max_speed = -1;
     Command *max_speed_command = NULL;
    while (!isEmpty(&bus_queue))
    {
         Command *command = dequeue(&bus_queue);
        // if the command can acquire the bus
        if (command->bus_speed > max_speed)
        {
            max_speed = command->bus_speed;
            max_speed_command = command;
        }
        enqueue(&temp_queue, command);
    }
    // push all the commands back to the bus queue
    while (!isEmpty(&temp_queue))
    {
         Command *command = dequeue(&temp_queue);
        if (command != max_speed_command)
        {
            enqueue(&bus_queue, command);
        }
    }
    // if  find a command that can acquire the bus
    if (max_speed_command != NULL)
    {
        // set the bus end time
        last_bus_end =total_SYS_time + TIME_ACQUIRE_BUS + ceil(max_speed_command->bus_size * 1000000.0 / max_speed_command->bus_speed);
        last_bus_command = max_speed_command;
    }
}

//---------------------------------- execute a systme call  -------------------------------
/**
 * @brief execute a syscall in certain command
 * @param command the command that is executing the syscall
 * @param syscall_index the index of the syscall in the command
 */
void execute_syscall( Command *command, int syscall_index)
{
   total_SYS_time += 1;
    switch (command->syscalls[syscall_index].type)
    {
    case READ:
        for (int i = 0; i < num_devices; i++)
        {
            if (strcmp(devices[i].device_name, command->syscalls[syscall_index].syscall_target) == 0)
            {
                command->bus_wait_start =total_SYS_time;
                command->bus_size = command->syscalls[syscall_index].io_size;
                command->bus_speed = devices[i].readspeed;
                break;
            }
        }
        enqueue(&bus_queue, command);
        update_state(command, Blocked);
        break;
    case WRITE:
        for (int i = 0; i < num_devices; i++)
        {
            if (strcmp(devices[i].device_name, command->syscalls[syscall_index].syscall_target) == 0)
            {
                command->bus_wait_start =total_SYS_time;
                command->bus_size = command->syscalls[syscall_index].io_size;
                command->bus_speed = devices[i].writespeed;
                break;
            }
        }
        enqueue(&bus_queue, command);
        update_state(command, Blocked);
        break;
    case SLEEP:
        command->start_sleep_time =total_SYS_time;
        command->sleep_duration = command->syscalls[syscall_index].sleep_time + 1;
        update_state(command, Blocked);
        enqueue(&sleep_queue, command);
        break;
    case WAIT:
        // check if all children have exited, if so we set the command to ready
        for (int i = 0; i < command->num_children; i++)
        {
            if (commands[command->child_index[i]].state != Exit)
            {
                update_state(command, Blocked);
                enqueue(&wait_queue, command);
                return;
            }
        }
        update_state(command, Ready);
        break;
    case EXIT:
        total_onCPU_time += command->syscalls[syscall_index].run_time;
        command->state = Exit;
        break;
    case SPAWN:
        for (int i = 0; i < num_commands; i++)
        {
            if (strcmp(commands[i].command_name, command->syscalls[syscall_index].spawn_command) == 0)
            {
                // copy the command to a new command, and enqueue it to the ready queue
                memcpy(&commands[num_commands], &commands[i], sizeof( Command));
                update_state(&commands[num_commands], Ready);
                commands[num_commands].elapsed_cpu_time = 0;
                commands[num_commands].executed_syscalls = 0;
                command->child_index[command->num_children++] = num_commands;
                num_commands++;
                update_state(command, Ready);
                break;
            }
        }
        break;
    default:
        break;
    }
}

//--------------------utility function for: execute_commands(void)------------------------------
/**
 * @brief check if all the processes have terminated
 * @return true if all the processes have exited, false otherwise
 */
bool process_terminated()
{
    int all_processes_terminated = true;
    for (int i = 0; i < num_commands; i++)
    {
        if (commands[i].state != Exit && commands[i].state != New)
        {
           all_processes_terminated = false;
            break;
        }
    }
    return all_processes_terminated;
}

//------------------------ handles the execution of commands-----------------------------------
void execute_commands(void)
{  
    num_commands++;
    // first set the state of the command to ready
    commands[0].state = Ready;
    // second enqueue the first command
    enqueue(&ready_queue, &commands[0]);
    // third start executing the commands
    while (!process_terminated())
    {
         Command *command = dequeue(&ready_queue);
        if (command != NULL)
        {
            update_state(command, Running);
            // then execute the command
            int syscall_index = command->executed_syscalls;
            // if the command can be executed in current time quantum
            if (command->syscalls[syscall_index].run_time < command->elapsed_cpu_time + time_quantum)
            {
               total_SYS_time += command->syscalls[syscall_index].run_time - command->elapsed_cpu_time;
                command->elapsed_cpu_time = command->syscalls[syscall_index].run_time;
                execute_syscall(command, syscall_index);
                command->executed_syscalls++;
            }
            else
            {
                //if the command cannot be executed in current time quantum, then enqueue it to the ready queue
                command->elapsed_cpu_time += time_quantum;
               total_SYS_time += time_quantum;
                update_state(command, Ready);
            }
        }
        // handle the sleep queue , the wait queue the bus queue
        handle_sleep_queue();
        handle_wait_queue();
        handle_bus_queue();

        if (process_terminated())
        {
            break;
        }
        // if the ready queue is empty, then increment the total system time
        if (isEmpty(&ready_queue))
        { 
           total_SYS_time += 1;
        }
    }
}

