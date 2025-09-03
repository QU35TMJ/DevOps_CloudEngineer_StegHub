##Linux - Chmod & Chown

###Understanding ownership and persmission in Linux.

###File Ownership

Every file and directory in Linux is associated with three key attributes:

-**Owner**: The user who created the file. This user has specific permissions to manage the file. 
-**Group**: A collection of users that can share access to the file. The group can have different permissions than the owner.
-**Others**: All other users on the system who are not the owner or part of the group.

###File Permission

Linux uses three basic types of permissions that can be assigned to the owner, group, and others:

-**Read (r)**: Allows users to view the contents of a file or list the contents of a directory.
-**Write (w)**: Allows users to modify a file's contents or add/remove files within a directory. 
-**Execute (x)**: Allows users to run a file as a program or script. For directories, it allows users to enter the directory.

In Linux, file permissions and ownership are crucial aspects of managing access to files and directories. Two important commands used for this purpose are `chmod` (change mode) and `chown` (change owner).

###`chmod` - Changing File Permissions

The chmod command is used to change the access permissions of files and directories. It allows you to specify who can read, write, or execute a file or directory.

```chmod [options] mode file```

*Arguments:*

1. **options:** These are flags that tells chmod what to do. some of these flags/options are:

-R: This option recursively apaplies/modifies permissions of directories and their contents.
-v: This flag tells chmod to display verbose informations on the changes being made to a file or directory in real time.
-c: This flag tells chmod to only display the information of only files whose permissions were altered.

2. **mode:** This spells out the permissions to be assigned to the file or directory and can be represented in three formats:

-*Symbolic mode:* In this format, letters (u, g, o, a), and symbols (+, -, =) are used to specify permissions (e.g: u+rwx, g-w, etc...). where u = user/owner, g = group, o = others, a = all, + = add permission(s), - = remove permission(s) E.g:

```chmod u+rwx, g=rw <filename>.txt```

This above example tells chmod to grant full access to the file's owner, only read/write access to the file's group.

-**Absolute mode:** This mode uses octal numbers (i.e: numbers between 0 to 7) to represent permissions. in this mode, read = 4, write = 2, and execute = 1. So assigning a permission to a file using this mode would look like:

```chmod 777 <filename>```

-**Reference mode:** This copies permissions from another file or directory and pastes it to a file.

###`chown` - Changing File Ownership

The chown command is used to change the owner and/or group ownership of files and directories. The basic syntax for chown is:

```chown [options] owner[:group] file```

**Arguments:**

1. **options:** This right here, controls how chmod works/behaves. some common options for chmod includes:

-R: This recursively changes the ownership of a direcctory and all their contents.
-v: This option tells chmod to display verbose information about the changes being made.
--reference: This option tells chmod to set the ownership of the file to match that of the referenced file (another file or another directory).

2. **new_owner:** This argument here is a place-holder for the username or the numeric user ID of the user who will become the new owner of the file or directory being modified.

3. **group:** This argument stipulates the new group (which can be stipulated via the group ID or the group name) that gets the ownership of the file or directory. E.g:

```chown alice:developers fileName.txt```

he above command tells chown to change the file's owner to 'alice' and the file group to 'developers'.

###Conclusion

The `chmod` and `chown` commands are very very important in ensuring the proper access control and data security of files in a Linux environment, and understanding these commands (as we have done in the above explanations) is crucial in managing file permissions, ownership and managing file data in the Linux environment.
