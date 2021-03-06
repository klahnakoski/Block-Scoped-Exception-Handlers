
Please excuse my nomenclature.  I hope the community can correct the synonyms that clarify my proposal.

Problem
-------

I program defensively, and surround many of my code blocks with try blocks to catch expected and unexpected errors.   Those unexpected errors seem to dominate in my code; I never really know how many ways my SQL library can fail, nor am I really sure that a particular key is in a `dict()`.   Most of the time I can do nothing about those unexpected errors; I simply chain them, with some extra description about what the code block was attempting to do.

I am using 2.7, so I have made my own convention for chaining exceptions.  3.x chains more elegantly:

    for t in todo:
        try:
            # do error prone stuff
        except Exception, e:
            raise ToDoError("oh dear!") from e

The “error prone stuff” can itself have more try blocks to catch known failure modes, maybe deal with them.  Add some `with` blocks and a conditional, and the nesting gets ugly:
 
    def process_todo(todo):
        try:
            with Timer("todo processing"):
                # pre-processing
                for t in todo:
                    try:
                        # do error prone stuff
                    except Exception, e:
                        raise TodoError("oh dear!") from e
                # post-processing
        except Exception, e:
            raise OverallTodoError("Not expected") from e        

Not only is my code dominated by exception handling, the meaningful code is deeply nested.


Solution
--------

I would like Python to have a bare `except` statement, which applies from statement, to the end of enclosing block.  Here is the same example using the new syntax:

    def process_todo(todo):
        except Exception, e:
            raise OverallTodoError("Not expected") from e
    
        with Timer("todo processing"):
            # pre-processing
            for t in todo:
                except Exception, e:
                    raise TodoError("oh dear!") from e

                # do error prone stuff
            # post-processing

Larger code blocks do a better job of portraying the visual impact of the reduced indentation.  I would admit that some readability is lost because the error handling code precedes the happy path, but I believe the eye will overlook this with little practice.

Compound `except` statements are allowed.  They apply as if they were used in a `try` statement; matched in the order declared:

    def process_todo(todo):
        pre_processing()  # has no exception handling

        except SQLException, e:  # effective until end of method
            raise Exception("Not expected") from e
        except Exception, e: 
            raise OverallTodoError("Oh dear!") from e

        processing() 

The same code block can have more than one `except` statement. 

    def process_todo(todo):
        pre_processing()  # no exception handling
   
        except SQLException, e:  # covers lines from here to beginning of next `except` statement
            raise Exception("Not expected") from e
        except Exception, e:   # catches other exception types
            raise Exception("Oh dear!") from e
   
        processing()  # Exceptions caught
   
        except SQLException, e:  # covers lines to end of method
            raise Exception("Happens, sometimes") from e
   
        post_processing()  # SQLException caught, but not Exception
   
In these cases, the main block is being partitioned.  Here is the same in legit Python:

    def process_todo(todo):
        pre_processing()  # no exception handling
   
        try:
            processing()  # Exceptions caught
        except SQLException, e:  # covers all lines from here to beginning of next except statement
            raise Exception("Not expected") from e
        except Exception, e:   # catches other exception types
            raise Exception("Oh dear!") from e
   
        try:
            post_processing()  # SQLException caught, but not Exception   
        except SQLException, e:  # covers a lines to end of method
            raise Exception("Happens, sometimes") from e
   
   
Other Thoughts
--------------

I only propose this for replacing `try` blocks that have no `else` or `finally` clause.  I am not limiting my proposal to exception chaining; Anything allowed in `except` clause would be allowed.


Q&A
---


### Postscript `except` Clause?
 
`except` clauses could be added to each of the major statement types (def, for, if, with, etc…).  which would make the example look like:

    def process_todo(todo):
        with Timer("todo processing"):
            # pre-processing
            for t in todo:
                # do error prone stuff
            except Exception, e:
                raise TodoError("oh dear!") from e
    
            # post-processing
    except Exception, e:
        raise OverallTodoError("Not expected") from e

But, I am suspicious this is more complicated than it looks to implement, and the `except` statement does seem visually detached from the block it applies to.

### Where does control resume if it doesn't raise?

Control would resume just before the end of the block, not after it,
much like in the case of a `try` statement:

    for t in todo:
        pre_process()

        except Exception, e:
            print "problem"+unicode(e)
   
        process()
   
Would be the same as

    for t in todo:
       pre_process()

       try:
           process()
       except Exception, e:
           print "problem"+unicode(e)
   
so, the control resumes at the end of the `for` block, but still inside
the loop   

### Using `with` Statement?

In the event that the `except` clause is simply chaining, we could leverage the `__exit__()` method to capture, annotate, chain and re-raise.   

    def process_todo(todo):
        with Except(OverallTodoError("Not expected")):
            with Timer("todo processing"):
                pre_process()
                for t in todo:
                    with Except(TodoError("oh dear!")):
                        process()
         
                post_process()    

### Can we `break` out of a loop?

We can leave an exception handler in multiple ways: With `return`, `raise`, `break` or `continue`.  The bare `except` clause is interpreted similarly in all cases, with respect to scope.  The `except` clause is defined in a block, and the exception handler may leave block in any legal way.  

    def process_todo(todo):
        pre_process()
        for t in todo:
            except Exception, e:
                break  # we can break out of a loop, allowed

            process()
        post_process()    

Is the same as:

    def process_todo(todo):
        pre_process()
        for t in todo:
			try:
	            process()
            except Exception, e:
                break
        post_process()    

The scope of `except` is natural with nested loops: 

    def process_todo(todo):
        pre_process()
        for t in todo:
            except Exception, e:
                break
			for u, v in t.items():
	            process()
        post_process()    

The `break` applies to the outer loop.  This can been seen in the expansion:

    def process_todo(todo):
        pre_process()
        for t in todo:
			try:
				for u, v in t.items():
		            process()
            except Exception, e:
                break
        post_process()    





















