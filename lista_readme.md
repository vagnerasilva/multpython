To save a list using AWS Systems Manager Parameter Store, you can store the list as a single string value within a parameter. You would then need to parse the string back into a list when retrieving it. The key is to define a structure within the string that clearly separates the individual items in the list.
Example:
1. Storing the list:
Create a parameter with a name, for example, /my_list.
Set the parameter's value to a string representation of your list. For example, if your list is ["apple", "banana", "cherry"], you could store it as "apple,banana,cherry".
You can also use a more robust separator like | if your list items might contain commas.
Use the aws ssm put-parameter command to store the parameter: aws ssm put-parameter --name /my_list --value "apple,banana,cherry".

2. Retrieving and parsing the list:
Use the aws ssm get-parameter command to retrieve the parameter's value: aws ssm get-parameter --name /my_list.
Parse the retrieved string value back into a list using your preferred programming language. For example, in Python, you could use split():

import json
import boto3

def get_parameter_list(parameter_name, region_name="us-east-1"):
    """
    Retrieves a parameter and parses its value as a list.

    Args:
        parameter_name (str): The name of the parameter.
        region_name (str, optional): The AWS region. Defaults to "us-east-1".

    Returns:
        list: A list of strings from the parameter value.
    """
    ssm = boto3.client("ssm", region_name=region_name)
    try:
        response = ssm.get_parameter(Name=parameter_name, WithDecryption=True)
        value = response["Parameter"]["Value"]
        # Parse the value as a list using the appropriate separator
        return value.split(",")
    except Exception as e:
        print(f"Error: {e}")
        return None

# Example usage:
my_list = get_parameter_list("/my_list")
if my_list:
    print(my_list)  # Output: ['apple', 'banana', 'cherry']



For storing potentially larger lists, consider using the advanced-parameter tier for increased capacity, as standard parameters have a 4KB size limit. 

https://aws.amazon.com/pt/blogs/mt/introducing-parameter-store-cross-account-sharing/
import json
import boto3

def lambda_handler(event, context):

    ssm = boto3.client('ssm')
    parameter = ssm.get_parameter(Name='arn:aws:ssm:[REGION]:[YOUR-ACCOUNT]:parameter/your-app/dev/appSettings')
    ssm_val = parameter['Parameter']['Value']

    return {
        'statusCode': 200,
        'body': json.dumps('SSM Param value in this account: ' + ssm_val)
    }

    ssm.get_parameter(Name='arn:aws:ssm:{region}:{account}:parameter/your-app/dev/appSettings '

https://aws.amazon.com/pt/blogs/mt/introducing-parameter-store-cross-account-sharing/
https://stackoverflow.com/questions/64373503/cross-account-access-to-ssm-parameters
