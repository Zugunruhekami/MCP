*Thinking...*

> Let's create a test class for the `DeepReportGraph` class. This test will:
> 
> 1. Instantiate the `DeepReportGraph` class
> 2. Set up necessary mocks for external services (LLM, search, etc.)
> 3. Run a query through the graph
> 4. Verify the results
> 
> Here's how we would structure the test:
> 
> 1. We'll use pytest as our testing framework
> 2. We'll use unittest.mock to mock external services
> 3. We'll set up configuration for the test
> 4. We'll create test methods for different aspects of the graph
> 
> Let's start by creating a test class for the `DeepReportGraph`:


# Testing the DeepReportGraph with LangGraph

Here's a comprehensive test class for the `DeepReportGraph` using pytest and LangGraph's testing utilities:

```python
import os
import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock, patch
from typing import Dict, List, Any, Optional
import json

from pydantic import BaseModel, Field
from langgraph.checkpoint import MemorySaver

# Assuming DeepReportGraph is in this path - adjust as needed
from app.graphs.deep_report import DeepReportGraph, DeepReportState, PlanSection, SearchResult, WebContent

class TestDeepReportGraph:
    """Test class for DeepReportGraph"""
    
    @pytest.fixture
    def config(self):
        """Fixture to provide test configuration"""
        return {
            "planner_provider": "openai",
            "planner_model": "gpt-4o",
            "writer_provider": "anthropic",
            "writer_model": "claude-3-5-sonnet",
            "search_provider": "tavily",
            "max_search_depth": 2,
            "token_budget": 6000,
            "number_of_queries": 2,
            "report_structure": {
                "intro": "Introduction to the topic",
                "body": "Main sections of research",
                "conclusion": "Summary of findings"
            }
        }
    
    @pytest.fixture
    def mock_llm_service(self):
        """Fixture to provide a mocked LLM service"""
        mock_service = AsyncMock()
        
        # Mock the structured generation for plan creation
        async def mock_plan(*args, **kwargs):
            return [
                PlanSection(
                    section_id="1",
                    title="Introduction",
                    type="introduction",
                    description="Overview of the topic",
                    search_queries=["topic overview", "topic history"]
                ),
                PlanSection(
                    section_id="2",
                    title="Main Section 1",
                    type="body",
                    description="First main section",
                    search_queries=["first aspect", "related concepts"]
                ),
                PlanSection(
                    section_id="3",
                    title="Conclusion",
                    type="conclusion",
                    description="Summary of key points",
                    search_queries=[]
                )
            ]
        
        # Mock the text generation for section writing
        async def mock_generate(*args, **kwargs):
            prompt = args[0] if args else kwargs.get("prompt", "")
            
            if "Introduction" in prompt:
                return "# Introduction\n\nThis is the introduction to the topic..."
            elif "Main Section" in prompt:
                return "# Main Section\n\nThis is the main section content..."
            elif "Conclusion" in prompt:
                return "# Conclusion\n\nIn conclusion, the key findings are..."
            else:
                return "Generic content for the prompt."
        
        mock_service.astructured_generate = AsyncMock(side_effect=mock_plan)
        mock_service.agenerate = AsyncMock(side_effect=mock_generate)
        
        return mock_service
    
    @pytest.fixture
    def mock_search_service(self):
        """Fixture to provide a mocked search service"""
        mock_service = AsyncMock()
        
        # Mock the search results
        async def mock_search(*args, **kwargs):
            query = args[0] if args else kwargs.get("query", "")
            
            return [
                SearchResult(
                    title=f"Result 1 for {query}",
                    url=f"https://example.com/1?q={query}",
                    snippet=f"This is the first search result snippet for {query}."
                ),
                SearchResult(
                    title=f"Result 2 for {query}",
                    url=f"https://example.com/2?q={query}",
                    snippet=f"This is the second search result snippet for {query}."
                )
            ]
        
        mock_service.asearch = AsyncMock(side_effect=mock_search)
        
        return mock_service
    
    @pytest.fixture
    def mock_content_service(self):
        """Fixture to provide a mocked content extraction service"""
        mock_service = AsyncMock()
        
        # Mock the content extraction
        async def mock_extract(*args, **kwargs):
            url = args[0] if args else kwargs.get("url", "")
            
            return WebContent(
                url=url,
                title=f"Content from {url}",
                content=f"This is the extracted content from {url}. It contains information relevant to the query."
            )
        
        mock_service.extract_content = AsyncMock(side_effect=mock_extract)
        
        return mock_service
    
    @pytest.fixture
    def deep_report_graph(self, config, mock_llm_service, mock_search_service, mock_content_service):
        """Fixture to provide a DeepReportGraph with mocked services"""
        # Create patches for service initialization methods
        with patch.object(DeepReportGraph, '_init_planner_llm_service', return_value=mock_llm_service), \
             patch.object(DeepReportGraph, '_init_writer_llm_service', return_value=mock_llm_service), \
             patch.object(DeepReportGraph, '_init_search_service', return_value=mock_search_service), \
             patch.object(DeepReportGraph, '_init_content_service', return_value=mock_content_service), \
             patch.object(DeepReportGraph, '_init_token_counter', return_value=MagicMock(count_tokens=lambda x: len(x) // 4)):
            
            # Create the graph with the mocked services
            graph = DeepReportGraph(config)
            
            # Replace the checkpointer with a memory saver for testing
            graph.checkpointer = MemorySaver()
            
            yield graph
    
    @pytest.mark.asyncio
    async def test_planning_phase(self, deep_report_graph):
        """Test the planning phase of the deep report workflow"""
        # Define a test query
        query = "The impact of artificial intelligence on healthcare"
        
        # Run the first phase (planning) only
        event_counter = 0
        plan_complete = False
        
        async for event in deep_report_graph.astream(query):
            event_counter += 1
            
            # Check event structure
            assert isinstance(event, dict)
            assert "current_phase" in event
            
            # Check if we've reached the plan review phase
            if event["current_phase"] == "plan_review" and not plan_complete:
                plan_complete = True
                
                # Verify the plan structure
                assert "plan" in event
                assert isinstance(event["plan"], list)
                assert len(event["plan"]) > 0
                
                # Check properties of the first plan section
                first_section = event["plan"][0]
                assert "section_id" in first_section
                assert "title" in first_section
                assert "type" in first_section
                assert "description" in first_section
                
                # Break after validating the plan
                break
        
        # Ensure events were generated
        assert event_counter > 0
        # Ensure we reached the plan review phase
        assert plan_complete is True
    
    @pytest.mark.asyncio
    async def test_full_report_generation(self, deep_report_graph):
        """Test the complete report generation process"""
        # Define a test query
        query = "The impact of artificial intelligence on healthcare"
        
        # Execute graph and collect events
        all_events = []
        plan_review_reached = False
        research_phase_reached = False
        writing_phase_reached = False
        completion_reached = False
        
        async for event in deep_report_graph.astream(query):
            all_events.append(event)
            
            # Track progress through phases
            if event["current_phase"] == "plan_review" and not plan_review_reached:
                plan_review_reached = True
                
                # Send a command to accept the plan and continue
                await deep_report_graph.send_feedback(True)
            
            if event["current_phase"] == "researching" and not research_phase_reached:
                research_phase_reached = True
                
                # Verify search operations
                assert "research_progress" in event
                assert "sections_complete" in event["research_progress"]
                assert "sections_total" in event["research_progress"]
            
            if event["current_phase"] == "writing" and not writing_phase_reached:
                writing_phase_reached = True
                
                # Verify writing operations
                assert "writing_progress" in event
                assert "sections_complete" in event["writing_progress"]
                assert "sections_total" in event["writing_progress"]
            
            if event["current_phase"] == "complete" and not completion_reached:
                completion_reached = True
                
                # Verify report was generated
                assert "final_report" in event
                assert isinstance(event["final_report"], str)
                assert len(event["final_report"]) > 0
        
        # Ensure all phases were reached
        assert plan_review_reached is True
        assert research_phase_reached is True
        assert writing_phase_reached is True
        assert completion_reached is True
    
    @pytest.mark.asyncio
    async def test_plan_feedback(self, deep_report_graph):
        """Test providing feedback to modify the plan"""
        # Define a test query
        query = "The impact of artificial intelligence on healthcare"
        
        # Execute graph until plan review
        plan_review_reached = False
        feedback_processed = False
        
        async for event in deep_report_graph.astream(query):
            # When plan review is reached, provide feedback
            if event["current_phase"] == "plan_review" and not plan_review_reached:
                plan_review_reached = True
                
                # Store the original plan
                original_plan = event["plan"]
                
                # Send feedback to modify the plan
                await deep_report_graph.send_feedback("Please add a section about ethical considerations")
            
            # Check if feedback was processed and new plan generated
            if event["current_phase"] == "plan_review" and plan_review_reached and not feedback_processed:
                feedback_processed = True
                
                # Verify the plan was updated
                updated_plan = event["plan"]
                assert updated_plan != original_plan
                
                # Accept the updated plan
                await deep_report_graph.send_feedback(True)
            
            # Once the plan is accepted, we can break
            if event["current_phase"] == "researching" and feedback_processed:
                break
        
        # Ensure feedback was processed
        assert plan_review_reached is True
        assert feedback_processed is True
    
    @pytest.mark.asyncio
    async def test_error_handling(self, deep_report_graph, mock_search_service):
        """Test error handling in the workflow"""
        # Define a test query
        query = "The impact of artificial intelligence on healthcare"
        
        # Make the search service occasionally fail
        mock_search_service.asearch.side_effect = [
            Exception("Search API error"),  # First call fails
            # Next calls succeed
            [
                SearchResult(
                    title="Result after error",
                    url="https://example.com/recovery",
                    snippet="This is a result after error recovery."
                )
            ]
        ]
        
        # Execute graph and check error handling
        error_occurred = False
        recovery_succeeded = False
        
        async for event in deep_report_graph.astream(query):
            # Check for error events
            if "errors" in event and event["errors"] and not error_occurred:
                error_occurred = True
                
                # Verify error information is present
                assert len(event["errors"]) > 0
                assert "message" in event["errors"][0]
                assert "step" in event["errors"][0]
            
            # Check for recovery
            if error_occurred and "current_phase" == "researching" and not recovery_succeeded:
                if "research_progress" in event and event["research_progress"]["sections_complete"] > 0:
                    recovery_succeeded = True
            
            # Break after verifying recovery
            if recovery_succeeded:
                break
        
        # Verify error handling and recovery
        assert error_occurred is True
        assert recovery_succeeded is True
```

## Adding Support Files

You'll need these additional files to make the test work:

### pytest.ini

```ini
[pytest]
asyncio_mode = auto
markers =
    asyncio: mark a test as an asyncio test
```

### conftest.py

```python
import os
import pytest
import asyncio

# Set environment variables for testing
os.environ["OPENAI_API_KEY"] = "test-api-key"
os.environ["ANTHROPIC_API_KEY"] = "test-api-key"
os.environ["TAVILY_API_KEY"] = "test-api-key"

# Configure pytest
def pytest_configure(config):
    config.addinivalue_line("markers", "asyncio: mark test as asyncio")

@pytest.fixture(scope="session")
def event_loop():
    """Create an event loop for the test session."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()
```

## Test Run Command

Run the tests with:

```bash
pytest tests/graphs/test_deep_report_graph.py -v
```

## Explanation of the Test Strategy

### 1. Test Fixtures

- **config**: Provides a standard test configuration
- **mock_llm_service**: Mocks the LLM services (for both planner and writer)
- **mock_search_service**: Mocks the search service
- **mock_content_service**: Mocks the content extraction service
- **deep_report_graph**: Creates a properly configured graph instance with all mocked dependencies

### 2. Test Methods

- **test_planning_phase**: Verifies the planning phase operates correctly
- **test_full_report_generation**: Tests the entire workflow from start to finish
- **test_plan_feedback**: Tests the human-in-the-loop feedback mechanism
- **test_error_handling**: Verifies that the system is robust against service failures

### 3. Mocking Strategy

The tests use `unittest.mock` to replace external dependencies, which:
- Prevents actual API calls during testing
- Allows controlled testing of error conditions
- Makes tests deterministic and repeatable
- Enables quick execution without external dependencies

### 4. Testing Async Workflows

- Tests are marked with `@pytest.mark.asyncio` to handle async functions
- The `astream` method is used to process events in real-time
- The test includes interaction with the graph (providing feedback) during execution

### 5. Assertions and Verification

Each test method includes specific assertions to verify:
- Event structure and content
- State transitions through different phases
- Error handling and recovery
- Completeness of the final output

This test suite provides comprehensive coverage of the `DeepReportGraph` functionality, ensuring that all parts of the workflow operate correctly and handle both normal operation and error conditions gracefully.
