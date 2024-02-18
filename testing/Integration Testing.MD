# Integration Testing: Exploring Different Approaches

Integration testing and unit testing play pivotal roles in ensuring the quality and reliability of software. While unit testing scrutinizes individual units of code in isolation, integration testing examines the interactions between various components or modules within a system.

## Exploring Common Approaches to Integration Testing

### Big-bang Testing

Big-bang testing is an integration strategy where all modules are integrated simultaneously and tested collectively, simulating the complete system.

#### Advantages:

* Simplicity: Big-bang testing is straightforward to implement as it involves integrating all modules at once and testing them together.

* Early Integration: It facilitates early integration of different components, enabling the detection of integration issues sooner in the development cycle.

#### Disadvantages:

* Complex Debugging: Identifying the root cause of failures can be challenging since all modules are integrated simultaneously, making it harder to isolate issues.

* Time-consuming and Expensive: Big-bang testing can be resource-intensive as it requires testing all modules together, potentially elongating the testing phase and increasing costs.

#### Example

**_NOTE:_**  This is a simple demonstration of the big-bang testing approach in Python.

```python
# account_management.py
class AccountManager:
    def create_account(self, account_id, initial_balance):
        # Logic to create an account
        return f"Account {account_id} created with balance {initial_balance}"


# transaction_processing.py
class TransactionProcessor:
    def process_transaction(self, account_id, amount):
        # Logic to process a transaction
        return f"Transaction of {amount} processed for account {account_id}"


# customer_support.py
class CustomerSupport:
    def resolve_issue(self, issue_id):
        # Logic to resolve a customer issue
        return f"Issue {issue_id} resolved"


# integration_test.py
import unittest

from account_management import AccountManager
from transaction_processing import TransactionProcessor
from customer_support import CustomerSupport


class IntegrationTest(unittest.TestCase):
    def test_integration(self):
        account_manager = AccountManager()
        transaction_processor = TransactionProcessor()
        customer_support = CustomerSupport()

        # Simulate interactions between modules
        account_creation_result = account_manager.create_account(12345, 1000)
        transaction_result = transaction_processor.process_transaction(12345, 500)
        issue_resolution_result = customer_support.resolve_issue(9876)

        # Assert results
        self.assertEqual(account_creation_result, "Account 12345 created with balance 1000")
        self.assertEqual(transaction_result, "Transaction of 500 processed for account 12345")
        self.assertEqual(issue_resolution_result, "Issue 9876 resolved")

if __name__ == '__main__':
    unittest.main()
```

### Top-down Testing

Top-down testing is a fundamental software testing methodology that initiates testing from the highest level of an application's architecture and systematically progresses downwards to lower levels.

#### Advantages:

* Early Issue Detection: Top-down testing enables the early identification of high-level issues, offering a holistic view of the system's functionality. This approach facilitates the testing of critical features at the outset of the development lifecycle.

* Holistic Understanding: By starting testing from the highest level of the application's architecture, top-down testing provides testers and stakeholders with a comprehensive understanding of the system's overall behavior and functionality. This broader perspective can aid in making informed decisions regarding system design, functionality prioritization, and overall project direction.

#### Disadvantages:

* Stub Dependency: A notable drawback of top-down testing is its reliance on stubs, which may not accurately replicate the behavior of lower-level modules. This can potentially lead to discrepancies between testing environments and production environments.

* Complexity with Stubs: As the number of testing stubs increases, the testing process can become intricate and convoluted. Managing and coordinating stubs across various levels of the application architecture can pose challenges, impacting the efficiency of the testing process.

#### Example

```python
# library_system.py

class BookManager:
    def __init__(self):
        self.books = []

    def add_book(self, title, author):
        book = {"title": title, "author": author}
        self.books.append(book)
        return True

class MemberManager:
    def __init__(self):
        self.members = []

    def add_member(self, name):
        member = {"name": name}
        self.members.append(member)
        return True

class LibrarySystem:
    def __init__(self):
        self.book_manager = BookManager()
        self.member_manager = MemberManager()

    def add_book(self, title, author):
        return self.book_manager.add_book(title, author)

    def add_member(self, name):
        return self.member_manager.add_member(name)

# Integration testing for LibrarySystem
import unittest

class TestLibrarySystemIntegration(unittest.TestCase):
    def setUp(self):
        self.library_system = LibrarySystem()

    def test_add_book(self):
        # Testing add_book method
        result = self.library_system.add_book("The Great Gatsby", "F. Scott Fitzgerald")
        self.assertTrue(result)
        self.assertEqual(len(self.library_system.book_manager.books), 1)
        self.assertEqual(self.library_system.book_manager.books[0]["title"], "The Great Gatsby")

    def test_add_member(self):
        # Testing add_member method
        result = self.library_system.add_member("John Doe")
        self.assertTrue(result)
        self.assertEqual(len(self.library_system.member_manager.members), 1)
        self.assertEqual(self.library_system.member_manager.members[0]["name"], "John Doe")

if __name__ == "__main__":
    unittest.main()
```

### Bottom-up Testing

Bottom-up testing integration is a systematic method where individual components are tested first, gradually integrating and testing larger modules from the bottom to the top of the system hierarchy. This approach facilitates early defect detection and fosters incremental development, ensuring a robust final product.

#### Advantages

* Efficient Troubleshooting: By not having to wait for all modules to be available for testing, bottom-up testing significantly reduces troubleshooting time. Testers can focus on individual components without being hindered by dependencies on other modules.

* Early Defect Detection: Starting testing at the lowest level allows for the early identification and rectification of defects within individual components. This proactive approach prevents issues from propagating to higher integration levels, ensuring a smoother development process.

### Disadvantages

* Delay in System-level Testing: While bottom-up integration testing prioritizes individual component integration, it may postpone system-level testing until later stages of development. This delay increases the risk of late detection of high-level integration issues or architectural flaws, potentially requiring extensive rework.

* Challenges in Simulating Real-world Interactions: Bottom-up integration testing may struggle to accurately simulate real-world interactions between components, especially when dependencies on higher-level modules or external systems exist. This can limit the effectiveness of testing in capturing system-wide behaviors and potential issues.

#### Example

```python
# module1.py
def add(a, b):
    return a + b

# module2.py
def multiply(a, b):
    return a * b

# test_module1.py - Unit tests for module1
import unittest
import module1

class TestModule1(unittest.TestCase):
    def test_add(self):
        self.assertEqual(module1.add(3, 5), 8)
        self.assertEqual(module1.add(0, 0), 0)
        self.assertEqual(module1.add(-3, 3), 0)

# test_module2.py - Unit tests for module2
import unittest
import module2

class TestModule2(unittest.TestCase):
    def test_multiply(self):
        self.assertEqual(module2.multiply(3, 5), 15)
        self.assertEqual(module2.multiply(0, 5), 0)
        self.assertEqual(module2.multiply(-3, 5), -15)

# integration_test.py - Bottom-up integration tests
import unittest
import module1
import module2

class TestIntegration(unittest.TestCase):
    def test_integration(self):
        self.assertEqual(module2.multiply(module1.add(2, 3), 4), 20)
        self.assertEqual(module2.multiply(module1.add(-2, 2), 5), 0)

if __name__ == "__main__":
    unittest.main()
```

### Sandwich/hybrid testing

The "Sandwich" or "Hybrid Testing" method merges elements from both top-down and bottom-up testing methodologies, creating a comprehensive testing strategy. It involves simultaneous testing at multiple levels of the software hierarchy, akin to layers in a sandwich.

#### Advantages

* Comprehensive Testing: Sandwich testing provides early defect detection, incremental integration, and thorough test coverage across various levels of the software hierarchy. By blending aspects of top-down and bottom-up approaches, it ensures a balanced testing strategy that enhances software quality and reliability.

### Disadvantages

* Increased Complexity: Managing sandwich testing requires meticulous planning and execution due to its inherent complexity. Coordinating testing efforts across different levels and ensuring collaboration between testing teams can be challenging.

* Dependency Management: Handling dependencies between components becomes crucial in sandwich testing. Ensuring that all components interact seamlessly while maintaining proper test coverage across all levels demands careful attention.

#### Example

```python
# Top-level functionality
class BankingSystem:
    def __init__(self, database):
        self.database = database
        self.accounts = {}

    def create_account(self, account_number, initial_balance):
        self.accounts[account_number] = initial_balance
        self.database.save_account(account_number, initial_balance)

    def deposit(self, account_number, amount):
        if account_number in self.accounts:
            self.accounts[account_number] += amount
            self.database.save_account(account_number, self.accounts[account_number])
        else:
            print("Account not found.")

    def withdraw(self, account_number, amount):
        if account_number in self.accounts:
            if self.accounts[account_number] >= amount:
                self.accounts[account_number] -= amount
                self.database.save_account(account_number, self.accounts[account_number])
            else:
                print("Insufficient funds.")
        else:
            print("Account not found.")

# Bottom-level functionality
class Database:
    def __init__(self):
        self.data = {}

    def save_account(self, account_number, balance):
        self.data[account_number] = balance

    def get_account_balance(self, account_number):
        return self.data.get(account_number, None)

# Test cases
import unittest

class TestBankingSystem(unittest.TestCase):
    def setUp(self):
        self.database = Database()
        self.banking_system = BankingSystem(self.database)

    def test_create_account(self):
        self.banking_system.create_account("123456", 100)
        self.assertEqual(self.database.get_account_balance("123456"), 100)

    def test_deposit(self):
        self.banking_system.create_account("789012", 50)
        self.banking_system.deposit("789012", 50)
        self.assertEqual(self.database.get_account_balance("789012"), 100)

    def test_withdraw(self):
        self.banking_system.create_account("345678", 100)
        self.banking_system.withdraw("345678", 50)
        self.assertEqual(self.database.get_account_balance("345678"), 50)

if __name__ == "__main__":
    unittest.main()
```

## Wiki

**Stubs** are snippet of code that replaces another component during testing.

# Fonts

* https://www.techtarget.com/searchsoftwarequality/definition/integration-testing#:~:text=Four%20key%20strategies%20to%20execute,them%20all%20as%20one%20unit.
* https://www.browserstack.com/guide/integration-testing
* https://www.geeksforgeeks.org/big-bang-integration-testing/
