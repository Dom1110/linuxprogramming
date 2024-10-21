# Instructional Lab: Understanding and Using Hard Links

## Introduction
This lab will guide you through understanding hard links in Unix-like systems and demonstrate their practical application in a real-world scenario.

## Part 1: Understanding Hard Links

Hard Links are different from symbolic links. Creating a hard link creates another file that points to the same underlying inode (the data structure that holds the file information) as the original file. This means that even if you delete the original file, the hard link will still work because it points to the same inode, which contains the data of the file.

To create a hard link, use the `ln` command without the `-s` option:

```
ln target_file link_name
```

Hard links can be useful when you want to have a single file appear in multiple directories, perhaps because different scripts expect to find the file in different locations, without having multiple separate copies of the file.

## Part 2: Practical Scenario - Database Connection File Shared Across Different Projects

In this scenario, we will:

1. Create a database connection file that contains the credentials to connect to a database.
2. Create hard links to the database connection file in different project directories.
3. Write Python scripts in each project directory to read the database credentials from the respective hard links to the database connection file.
4. Update the database connection file to point to a new database, and demonstrate that all projects automatically use the new database credentials without requiring any changes to the individual project scripts.

### Step 1: Create the database configuration file

Create a file named `db_config.json` with the following content:

```json
{
    "host": "initial_db_host",
    "user": "initial_db_user",
    "password": "initial_db_password"
}
```

### Step 2: Create hard links in different project directories

```bash
mkdir project1 project2
ln db_config.json project1/db_config.json
ln db_config.json project2/db_config.json
```

### Step 3: Create Python scripts to read the database credentials

Create two identical Python scripts: `project1/script.py` and `project2/script.py` with the following content:

```python
# project1/script.py and project2/script.py
import json

def get_db_credentials():
    with open("db_config.json", 'r') as file:
        credentials = json.load(file)
    return credentials

credentials = get_db_credentials()
print(f"Connecting to database with the following credentials: {credentials}")
```

### Step 4: Update the database configuration file

Update the `db_config.json` file with new credentials:

```json
{
    "host": "new_db_host",
    "user": "new_db_user",
    "password": "new_db_password"
}
```

### Step 5: Run the scripts and observe the results

Run both scripts:

```bash
python project1/script.py
python project2/script.py
```

Both scripts will output:

```
Connecting to database with the following credentials: {'host': 'new_db_host', 'user': 'new_db_user', 'password': 'new_db_password'}
```

## Discussion

In this scenario, using hard links allows the different projects to always use the latest database credentials without having to update the credentials in multiple places. It ensures a single source of truth for the database credentials, thereby simplifying maintenance.

Comparing this with symbolic (soft) links, both could work in this scenario. However, hard links have an advantage in that they allow the file to remain accessible even if the original file is deleted, whereas symbolic links would break if the original file is deleted. Moreover, hard links ensure that the file exists in all linked directories as if it were a normal file, not a link, providing an additional level of transparency to the scripts accessing the file.

## Potential Pitfall

If you modify the content of a file through a hard link, it will affect all other hard links to that file, including what would be perceived as the "original" file. This is because all the hard links to a file actually point to the same inode on the filesystem, which is where the actual file data is stored. Any change made through one hard link is reflected in all other hard links because they all share the same inode reference.

In scenarios where you want a single source of truth and you want changes to be propagated to all instances of a file automatically, hard links can be very beneficial. For example, in the database credentials example, it ensures that all your projects are always using the latest database credentials without needing to update each one individually.

But it can indeed be a disadvantage in situations where you don't want changes to affect all instances of a file. It can potentially lead to unexpected behaviors if a person or a script modifies a file not realizing that it is a hard link and that the changes will affect other hard links. This risk can sometimes be mitigated by documenting the setup clearly and ensuring that all developers or users who might interact with the files are aware of how hard links work and what the implications of changing a hard link are.

In practice, when using hard links, it is often done in a controlled manner with scripts or applications managing the links to prevent unintended alterations from occurring.

## Advanced Scenario: Safe Update with Controlled Management of Hard Links

In this scenario, we:

1. Initialize a central configuration file with default settings.
2. Create hard links to this file in different project directories.
3. Implement a function that performs controlled updates to the configuration file, temporarily lifting the read-only restrictions to apply the necessary changes and then reinstating the restrictions immediately after.

Here is a Python script that implements this scenario:

```python
import json
import os

# Path to the central configuration file
central_config_path = "central_config.json"

def initialize_config():
    """Initialize the central configuration file with default settings."""
    config_data = {
        "database": {
            "host": "default_host",
            "user": "default_user",
            "password": "default_password",
        }
    }
    
    with open(central_config_path, 'w') as file:
        json.dump(config_data, file, indent=4)

def create_hard_links():
    """Create hard links to the central configuration file in different project directories."""
    project_dirs = ["project1", "project2"]
    
    for project_dir in project_dirs:
        os.makedirs(project_dir, exist_ok=True)
        os.link(central_config_path, os.path.join(project_dir, "config.json"))

def update_config(key, value):
    """Update a configuration setting in a controlled manner."""
    with open(central_config_path, 'r') as file:
        config_data = json.load(file)
    
    keys = key.split(".")
    temp = config_data
    for k in keys[:-1]:
        temp = temp[k]
    temp[keys[-1]] = value
    
    with open(central_config_path, 'w') as file:
        json.dump(config_data, file, indent=4)

def safe_update(key, value):
    """Perform a safe update, temporarily lifting the read-only restriction."""
    # Temporarily grant write permissions
    os.system(f"chmod 644 {central_config_path}")

    # Perform the update
    try:
        update_config(key, value)
        print("Update successful.")
    except Exception as e:
        print(f"Update failed: {e}")

    # Revoke write permissions to maintain safety
    os.system(f"chmod 444 {central_config_path}")

# Initialize the central configuration file with default settings
initialize_config()

# Create hard links to the central configuration file in different project directories
create_hard_links()

# Set the file as read-only to prevent unauthorized changes
os.system(f"chmod 444 {central_config_path}")

# Perform a safe update using the controlled function
safe_update("database.host", "new_secure_host")
```

### Explanation of the script:

1. We define a global variable to hold the path to the central configuration file.
2. The `initialize_config` and `create_hard_links` functions serve to initialize the configuration file and create hard links to it in different project directories, respectively.
3. The `update_config` function allows us to update a configuration setting in a controlled manner.
4. The `safe_update` function is where we ensure a safe update environment. It:
   - Temporarily grants write permissions to the central configuration file.
   - Calls `update_config` to perform the update, with error handling to catch and report any issues that occur during the update.
   - Revokes write permissions to return the file to a read-only state, preventing unauthorized changes.
5. After initializing the configuration file and creating hard links, we set the file to read-only using the `chmod 444` command.
6. Finally, we perform a safe update using our controlled function, demonstrating that we can still update the file while maintaining a read-only restriction to prevent unauthorized changes.

This script implements a "Safe Update" strategy, allowing updates to be performed in a controlled manner while maintaining strict security through read-only file permissions.

## Recap

### Soft Links (Symbolic Links)

- **Creation**: Created using `ln -s source_file link_name`.
- **Behavior**: Point to the location of another file. If the original file is deleted, the soft link breaks.
- **Usage Scenarios**: Can link files across different filesystems, can link to directories, and are useful where the link and the original file need to be distinct entities, for instance in dynamically changing environments.

### Hard Links

- **Creation**: Created using `ln source_file link_name`.
- **Behavior**: Point to the inode of the original file, essentially creating another entry to the same file content. Any change in one hard link is reflected across all hard links. If the original filename is deleted, the content remains accessible through the hard links.
- **Usage Scenarios**: Useful in situations where you want a single file to appear in multiple directories without maintaining separate copies. It's leveraged in scenarios where a unified update across all instances is desired, ensuring a single source of truth.
