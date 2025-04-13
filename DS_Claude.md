*Thinking...*

> I need to compare and summarize the similarities and key differences between three implementations of deep search features similar to those offered by OpenAI, Perplexity, and Grok. Then I'll design a combined implementation using LangGraph framework.
> 
> Let's start by extracting the key features and characteristics of each implementation:
> 
> ## 1. Hugging Face's SmolaAgents Open Deep Research
> 
> From the GitHub URL: https://github.com/huggingface/smolagents/tree/main/examples/open_deep_research
> 
> Key features and characteristics:
> - Aims to replicate OpenAI's Deep Research model
> - Achieves 55% pass@1 on the GAIA validation set (compared to 67% for the original Deep Research)
> - Uses the GoogleSearchTool for web search with different provider options (SerpApi, Serper)
> - Requires API keys for search providers and models
> - Implemented in Python
> - Mentioned default model is "o1" which requires tier-3 access
> - Simple CLI interface
> 
> ## 2. Jina AI's node-DeepResearch
> 
> From the GitHub URL: https://github.com/jina-ai/node-DeepResearch
> 
> Key features and characteristics:
> - Focuses on finding the right answers via an iterative process
> - Explicitly states it doesn't optimize for long-form articles (unlike OpenAI/Gemini/Perplexity)
> - Implements a loop of Search → Read → Reason until budget is exceeded
> - Uses Gemini (gemini-2.0-flash) / OpenAI / LocalLLM for reasoning
> - Uses Jina Reader for searching and reading webpages
> - Implemented in TypeScript (Node.js)
> - Provides OpenAI-compatible Server API
> - Offers Docker setup
> - Has extensive UI and visualization capabilities
> - Offers both a hosted service and local deployment options
> - Has support for streaming responses
> 
> ## 3. LangChain's Open Deep Research
> 
> From the GitHub URL: https://github.com/langchain-ai/open_deep_research
> 
> Key features and characteristics:
> - Automates research and produces customizable reports on any topic
> - Uses LangGraph for workflow orchestration
> - Offers customization of research and writing process (models, prompts, report structure, search tools)
> - Supports various search tools (Tavily, Perplexity, Exa, ArXiv, PubMed, Linkup, DuckDuckGo, Google)
> - Uses a planner LLM and a writer LLM for different phases
> - Follows a plan-and-execute workflow with human-in-the-loop approval
> - Research and writing are done in parallel for different sections
> - Supports local deployment and hosted deployment via LangGraph Platform
> - UI through LangGraph Studio
> - Implemented in Python
> 
> Now, let's analyze the similarities and differences:
> 
> ## Similarities:
> 1. All implement some form of deep search or research capabilities
> 2. All use LLMs as the reasoning engine
> 3. All use external search tools for web content retrieval
> 4. All require API keys for LLMs and search providers
> 5. All support local deployment options
> 
> ## Key Differences:
> 
> ### Implementation Language:
> - SmolaAgents: Python
> - Jina AI: TypeScript (Node.js)
> - LangChain: Python with LangGraph
> 
> ### Architecture and Process Flow:
> - SmolaAgents: Seems to be a more direct replication of OpenAI's Deep Research model
> - Jina AI: Implements a specific iterative loop (Search → Read → Reason) with a focus on finding answers
> - LangChain: Uses a plan-and-execute workflow with separate planning and research phases
> 
> ### Output Format:
> - SmolaAgents: Focused on answering questions directly
> - Jina AI: Focused on concise answers from deep search, explicitly not for long-form content
> - LangChain: Produces customizable reports with structured sections
> 
> ### Customization and Flexibility:
> - SmolaAgents: Limited customization mentioned in the readme
> - Jina AI: Offers customization through config and different LLM options
> - LangChain: Highly customizable with options for report structure, search depth, model selection, etc.
> 
> ### User Interface:
> - SmolaAgents: Simple CLI interface
> - Jina AI: Offers UI visualization and OpenAI-compatible API
> - LangChain: Uses LangGraph Studio UI
> 
> ### Human-in-the-loop:
> - SmolaAgents: Not explicitly mentioned
> - Jina AI: Not explicitly mentioned for the reasoning process
> - LangChain: Explicitly supports human-in-the-loop approval of report plans
> 
> ### Search Tool Integration:
> - SmolaAgents: Limited to GoogleSearchTool with different providers
> - Jina AI: Uses Jina Reader for search and reading
> - LangChain: Supports a wide range of search tools (Tavily, Perplexity, Exa, ArXiv, PubMed, etc.)
> 
> Now, I'll design an approach to combine these implementations using the LangGraph framework:
> 
> ## Combined Implementation Design
> 
> ### Architecture Overview:
> 
> I'll design a system that combines the best aspects of all three implementations:
> 1. The focused answer-finding approach of Jina AI
> 2. The customizable report generation of LangChain
> 3. The performance benchmarking approach of SmolaAgents
> 
> The architecture will consist of:
> 
> 1. **Core Engine**: LangGraph-based workflow manager
> 2. **Search Module**: Unified interface to multiple search providers
> 3. **Reasoning Module**: LLM-based reasoning with configurable models
> 4. **UI Layer**: React-based frontend for interaction and visualization
> 5. **API Layer**: FastAPI backend providing OpenAI-compatible endpoints
> 
> ### Workflow Design:
> 1. **Query Understanding**: Parse and understand the user query
> 2. **Mode Selection**: 
>    - Quick Answer Mode (like Jina AI)
>    - Deep Report Mode (like LangChain)
>    - Research Mode (combine aspects)
> 
> 3. **Planning Phase**:
>    - Generate initial search queries
>    - For Report Mode: Create a report outline for user approval
> 
> 4. **Research Phase**:
>    - Iterative search → read → reason loop (from Jina AI)
>    - Parallel research for different sections (from LangChain)
>    - Token budget management (from Jina AI)
> 
> 5. **Synthesis Phase**:
>    - For Quick Answer: Direct concise response with citations
>    - For Report: Structured report with sections, citations, references
>    - For Research: Detailed findings with evidence and confidence levels
> 
> 6. **Feedback Loop**:
>    - Allow user feedback at multiple stages
>    - Support for redirecting the research based on feedback
> 
> ### Technology Stack:
> - **Backend**: Python with FastAPI, LangGraph for workflow orchestration
> - **Frontend**: JavaScript/React
> - **Models**: Support for multiple LLM providers (OpenAI, Anthropic, open-source models)
> - **Search**: Unified API for multiple search providers
> - **Deployment**: Docker containers for easy deployment
> - **Monitoring**: Telemetry and performance tracking
> 
> ### Project Structure:
> 
> ```
> deep_research/
> ├── backend/
> │   ├── app/
> │   │   ├── api/
> │   │   │   ├── __init__.py
> │   │   │   ├── routes.py
> │   │   │   └── endpoints/
> │   │   │       ├── __init__.py
> │   │   │       ├── chat.py
> │   │   │       ├── research.py
> │   │   │       └── report.py
> │   │   ├── core/
> │   │   │   ├── __init__.py
> │   │   │   ├── config.py
> │   │   │   └── settings.py
> │   │   ├── graphs/
> │   │   │   ├── __init__.py
> │   │   │   ├── base.py
> │   │   │   ├── quick_answer.py
> │   │   │   ├── deep_report.py
> │   │   │   └── research.py
> │   │   ├── models/
> │   │   │   ├── __init__.py
> │   │   │   └── schemas.py
> │   │   ├── services/
> │   │   │   ├── __init__.py
> │   │   │   ├── llm/
> │   │   │   │   ├── __init__.py
> │   │   │   │   ├── base.py
> │   │   │   │   ├── openai.py
> │   │   │   │   ├── anthropic.py
> │   │   │   │   └── local.py
> │   │   │   ├── search/
> │   │   │   │   ├── __init__.py
> │   │   │   │   ├── base.py
> │   │   │   │   ├── tavily.py
> │   │   │   │   ├── perplexity.py
> │   │   │   │   ├── exa.py
> │   │   │   │   ├── arxiv.py
> │   │   │   │   └── pubmed.py
> │   │   │   └── content/
> │   │   │       ├── __init__.py
> │   │   │       ├── reader.py
> │   │   │       └── parser.py
> │   │   ├── utils/
> │   │   │   ├── __init__.py
> │   │   │   ├── logging.py
> │   │   │   └── helpers.py
> │   │   └── main.py
> │   ├── tests/
> │   │   ├── __init__.py
> │   │   ├── test_api.py
> │   │   ├── test_graphs.py
> │   │   └── test_services.py
> │   ├── Dockerfile
> │   ├── requirements.txt
> │   └── pyproject.toml
> ├── frontend/
> │   ├── public/
> │   │   ├── index.html
> │   │   └── favicon.ico
> │   ├── src/
> │   │   ├── assets/
> │   │   ├── components/
> │   │   │   ├── Chat/
> │   │   │   ├── Research/
> │   │   │   ├── Report/
> │   │   │   ├── Visualization/
> │   │   │   └── common/
> │   │   ├── contexts/
> │   │   ├── hooks/
> │   │   ├── pages/
> │   │   ├── services/
> │   │   ├── styles/
> │   │   ├── utils/
> │   │   ├── App.jsx
> │   │   ├── index.jsx
> │   │   └── routes.jsx
> │   ├── .eslintrc.js
> │   ├── package.json
> │   ├── vite.config.js
> │   └── Dockerfile
> ├── docker-compose.yml
> ├── .env.example
> ├── README.md
> └── LICENSE
> ```
> 
> ### Key Components and Files:
> 
> #### Backend:
> 
> 1. **app/main.py**: Main FastAPI application entry point
> ```python
> from fastapi import FastAPI
> from fastapi.middleware.cors import CORSMiddleware
> from app.api.routes import router
> from app.core.config import settings
> 
> app = FastAPI(
>     title="Deep Research API",
>     description="Combined implementation of deep research features",
>     version="1.0.0"
> )
> 
> # Configure CORS
> app.add_middleware(
>     CORSMiddleware,
>     allow_origins=settings.CORS_ORIGINS,
>     allow_credentials=True,
>     allow_methods=["*"],
>     allow_headers=["*"],
> )
> 
> # Include API routes
> app.include_router(router)
> 
> if __name__ == "__main__":
>     import uvicorn
>     uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
> ```
> 
> 2. **app/core/config.py**: Configuration settings
> ```python
> from pydantic import BaseSettings, validator
> from typing import List, Optional
> import os
> 
> class Settings(BaseSettings):
>     # API configuration
>     API_V1_STR: str = "/api/v1"
>     PROJECT_NAME: str = "Deep Research"
>     
>     # CORS
>     CORS_ORIGINS: List[str] = ["http://localhost:3000", "http://localhost:5173"]
>     
>     # LLM API keys
>     OPENAI_API_KEY: Optional[str] = None
>     ANTHROPIC_API_KEY: Optional[str] = None
>     COHERE_API_KEY: Optional[str] = None
>     
>     # Search API keys
>     TAVILY_API_KEY: Optional[str] = None
>     PERPLEXITY_API_KEY: Optional[str] = None
>     EXA_API_KEY: Optional[str] = None
>     SERPER_API_KEY: Optional[str] = None
>     SERPAPI_API_KEY: Optional[str] = None
>     
>     # Default models
>     DEFAULT_PLANNER_PROVIDER: str = "openai"
>     DEFAULT_PLANNER_MODEL: str = "gpt-4-turbo"
>     DEFAULT_WRITER_PROVIDER: str = "anthropic"
>     DEFAULT_WRITER_MODEL: str = "claude-3-opus"
>     
>     # Default search provider
>     DEFAULT_SEARCH_PROVIDER: str = "tavily"
>     
>     # Research settings
>     DEFAULT_MAX_SEARCH_DEPTH: int = 2
>     DEFAULT_MAX_SEARCHES_PER_SECTION: int = 3
>     DEFAULT_TOKEN_BUDGET: int = 100000
>     
>     # Application settings
>     ENABLE_TELEMETRY: bool = True
>     LOG_LEVEL: str = "INFO"
>     
>     class Config:
>         env_file = ".env"
>         case_sensitive = True
> 
> settings = Settings()
> ```
> 
> 3. **app/graphs/base.py**: Base LangGraph class for all workflows
> ```python
> from langgraph.graph import StateGraph
> from typing import Dict, Any, TypedDict, Optional, List
> from pydantic import BaseModel
> 
> class BaseState(TypedDict):
>     query: str
>     context: List[Dict[str, Any]]
>     intermediate_results: List[Dict[str, Any]]
>     final_answer: Optional[str]
>     config: Dict[str, Any]
>     errors: List[Dict[str, Any]]
> 
> class BaseGraph:
>     """Base class for all LangGraph workflows"""
>     
>     def __init__(self, config: Dict[str, Any] = None):
>         self.config = config or {}
>         self.graph = self._build_graph()
>     
>     def _build_graph(self) -> StateGraph:
>         """Build the core graph structure - to be implemented by subclasses"""
>         raise NotImplementedError
>     
>     def compile(self):
>         """Compile the graph"""
>         return self.graph.compile()
>     
>     async def arun(self, query: str, config: Dict[str, Any] = None) -> Dict[str, Any]:
>         """Run the graph asynchronously"""
>         state = {
>             "query": query,
>             "context": [],
>             "intermediate_results": [],
>             "final_answer": None,
>             "config": self.config | (config or {}),
>             "errors": []
>         }
>         graph = self.compile()
>         result = await graph.ainvoke(state)
>         return result
>     
>     async def astream(self, query: str, config: Dict[str, Any] = None):
>         """Stream results from the graph asynchronously"""
>         state = {
>             "query": query,
>             "context": [],
>             "intermediate_results": [],
>             "final_answer": None,
>             "config": self.config | (config or {}),
>             "errors": []
>         }
>         graph = self.compile()
>         async for event in graph.astream(state):
>             yield event
> ```
> 
> 4. **app/graphs/quick_answer.py**: Quick Answer workflow (Jina AI style)
> ```python
> from langgraph.graph import StateGraph
> from typing import Dict, Any, List, Literal, TypedDict, Optional, Union, Annotated
> from pydantic import BaseModel, Field
> from app.graphs.base import BaseGraph, BaseState
> from app.services.llm import get_llm
> from app.services.search import get_search_provider
> from app.services.content import fetch_content
> import uuid
> 
> class SearchAction(BaseModel):
>     action: Literal["search"] = "search"
>     query: str
> 
> class VisitAction(BaseModel):
>     action: Literal["visit"] = "visit"
>     urls: List[str]
> 
> class AnswerAction(BaseModel):
>     action: Literal["answer"] = "answer"
>     answer: str
> 
> class ThinkAction(BaseModel):
>     action: Literal["think"] = "think"
>     thoughts: str
>     
> Action = Annotated[
>     Union[SearchAction, VisitAction, AnswerAction, ThinkAction],
>     Field(discriminator="action")
> ]
> 
> class QuickAnswerState(BaseState):
>     budget_used: int = 0
>     max_budget: int = 100000
>     urls_visited: List[str] = []
>     urls_to_visit: List[str] = []
>     search_queries: List[str] = []
>     
> class QuickAnswerGraph(BaseGraph):
>     """Quick Answer workflow - focused on finding answers efficiently"""
>     
>     def _build_graph(self) -> StateGraph:
>         workflow = StateGraph(QuickAnswerState)
>         
>         # Define nodes
>         workflow.add_node("decide_next_action", self._decide_next_action)
>         workflow.add_node("perform_search", self._perform_search)
>         workflow.add_node("visit_url", self._visit_url)
>         workflow.add_node("generate_answer", self._generate_answer)
>         workflow.add_node("think", self._think)
>         
>         # Define edges
>         workflow.set_entry_point("decide_next_action")
>         
>         workflow.add_conditional_edges(
>             "decide_next_action",
>             self._route_action,
>             {
>                 "search": "perform_search",
>                 "visit": "visit_url",
>                 "answer": "generate_answer",
>                 "think": "think"
>             }
>         )
>         
>         workflow.add_edge("perform_search", "decide_next_action")
>         workflow.add_edge("visit_url", "decide_next_action")
>         workflow.add_edge("think", "decide_next_action")
>         
>         # Define exit
>         workflow.set_finish_point("generate_answer")
>         
>         return workflow
>     
>     def _route_action(self, state: QuickAnswerState) -> str:
>         # Check if we've exceeded the budget
>         if state["budget_used"] >= state["max_budget"]:
>             return "answer"
>             
>         action = state.get("action", {})
>         return action.get("action", "search")
>     
>     async def _decide_next_action(self, state: QuickAnswerState) -> Dict[str, Any]:
>         """Decide what to do next"""
>         config = state["config"]
>         llm = get_llm(
>             provider=config.get("reasoning_provider", "openai"),
>             model=config.get("reasoning_model", "gpt-4o")
>         )
>         
>         # Create prompt with all context
>         prompt = self._create_reasoning_prompt(state)
>         
>         # Get next action
>         response = await llm.astructured_generate(
>             prompt, 
>             response_model=Action,
>             temperature=0.2
>         )
>         
>         return {"action": response}
>     
>     async def _perform_search(self, state: QuickAnswerState) -> Dict[str, Any]:
>         """Execute search with the given query"""
>         config = state["config"]
>         action = state["action"]
>         
>         search_provider = get_search_provider(
>             provider=config.get("search_provider", "tavily"),
>             api_key=config.get(f"{config.get('search_provider', 'tavily').upper()}_API_KEY")
>         )
>         
>         search_results = await search_provider.asearch(action.query)
>         
>         # Update state
>         new_urls = [result["url"] for result in search_results if result["url"] not in state["urls_visited"]]
>         
>         return {
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "search", "query": action.query, "results": search_results}
>             ],
>             "urls_to_visit": state["urls_to_visit"] + new_urls,
>             "search_queries": state["search_queries"] + [action.query],
>             "budget_used": state["budget_used"] + 100  # Approximate token cost
>         }
>     
>     async def _visit_url(self, state: QuickAnswerState) -> Dict[str, Any]:
>         """Visit URLs and extract content"""
>         action = state["action"]
>         urls = action.urls[:3]  # Limit to 3 URLs at a time
>         
>         results = []
>         for url in urls:
>             if url in state["urls_visited"]:
>                 continue
>                 
>             content = await fetch_content(url)
>             results.append({"url": url, "content": content})
>         
>         return {
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "visit", "urls": urls, "content": results}
>             ],
>             "urls_visited": state["urls_visited"] + urls,
>             "urls_to_visit": [u for u in state["urls_to_visit"] if u not in urls],
>             "budget_used": state["budget_used"] + sum(len(r["content"]) // 4 for r in results)  # Approximate token cost
>         }
>     
>     async def _think(self, state: QuickAnswerState) -> Dict[str, Any]:
>         """Process information and update context"""
>         action = state["action"]
>         
>         return {
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "thinking", "thoughts": action.thoughts}
>             ],
>             "budget_used": state["budget_used"] + len(action.thoughts) // 4  # Approximate token cost
>         }
>     
>     async def _generate_answer(self, state: QuickAnswerState) -> Dict[str, Any]:
>         """Generate final answer"""
>         config = state["config"]
>         action = state["action"]
>         llm = get_llm(
>             provider=config.get("reasoning_provider", "openai"),
>             model=config.get("reasoning_model", "gpt-4o")
>         )
>         
>         # If we already have an answer from the action, use it
>         if hasattr(action, "answer") and action.answer:
>             return {
>                 "final_answer": action.answer,
>                 "intermediate_results": state["intermediate_results"] + [
>                     {"type": "answer", "answer": action.answer}
>                 ]
>             }
>         
>         # Otherwise generate a best-effort answer based on the information collected
>         prompt = self._create_answer_prompt(state)
>         answer = await llm.agenerate(prompt)
>         
>         return {
>             "final_answer": answer,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "answer", "answer": answer}
>             ]
>         }
>     
>     def _create_reasoning_prompt(self, state: QuickAnswerState) -> str:
>         """Create prompt for the reasoning step"""
>         # Implementation would go here
>         pass
>     
>     def _create_answer_prompt(self, state: QuickAnswerState) -> str:
>         """Create prompt for the final answer generation"""
>         # Implementation would go here
>         pass
> ```
> 
> 5. **app/graphs/deep_report.py**: Deep Report workflow (LangChain style)
> ```python
> from langgraph.graph import StateGraph
> from typing import Dict, Any, List, Literal, TypedDict, Optional, Union, Annotated
> from pydantic import BaseModel, Field
> from app.graphs.base import BaseGraph, BaseState
> from app.services.llm import get_llm
> from app.services.search import get_search_provider
> from app.services.content import fetch_content
> import uuid
> 
> class ReportSection(BaseModel):
>     title: str
>     content: Optional[str] = None
>     status: str = "planned"  # planned, researching, writing, completed
>     research_queries: List[str] = []
>     research_results: List[Dict[str, Any]] = []
> 
> class ReportPlan(BaseModel):
>     title: str
>     sections: List[ReportSection]
>     introduction: Optional[str] = None
>     conclusion: Optional[str] = None
> 
> class DeepReportState(BaseState):
>     report_plan: Optional[ReportPlan] = None
>     awaiting_user_feedback: bool = False
>     user_feedback: Optional[str] = None
>     current_section_index: Optional[int] = None
> 
> class DeepReportGraph(BaseGraph):
>     """Deep Report workflow - focused on creating comprehensive research reports"""
>     
>     def _build_graph(self) -> StateGraph:
>         workflow = StateGraph(DeepReportState)
>         
>         # Define nodes
>         workflow.add_node("plan_report", self._plan_report)
>         workflow.add_node("wait_for_user_feedback", self._wait_for_user_feedback)
>         workflow.add_node("process_user_feedback", self._process_user_feedback)
>         workflow.add_node("select_next_section", self._select_next_section)
>         workflow.add_node("research_section", self._research_section)
>         workflow.add_node("write_section", self._write_section)
>         workflow.add_node("write_introduction", self._write_introduction)
>         workflow.add_node("write_conclusion", self._write_conclusion)
>         workflow.add_node("finalize_report", self._finalize_report)
>         
>         # Define edges
>         workflow.set_entry_point("plan_report")
>         
>         workflow.add_edge("plan_report", "wait_for_user_feedback")
>         
>         workflow.add_conditional_edges(
>             "wait_for_user_feedback",
>             self._check_user_feedback,
>             {
>                 "approved": "select_next_section",
>                 "feedback": "process_user_feedback"
>             }
>         )
>         
>         workflow.add_edge("process_user_feedback", "plan_report")
>         workflow.add_edge("select_next_section", "research_section")
>         workflow.add_edge("research_section", "write_section")
>         
>         workflow.add_conditional_edges(
>             "write_section",
>             self._check_sections_status,
>             {
>                 "next_section": "select_next_section",
>                 "introduction": "write_introduction",
>                 "all_done": "write_conclusion",
>             }
>         )
>         
>         workflow.add_edge("write_introduction", "write_conclusion")
>         workflow.add_edge("write_conclusion", "finalize_report")
>         
>         # Define exit
>         workflow.set_finish_point("finalize_report")
>         
>         return workflow
>     
>     def _check_user_feedback(self, state: DeepReportState) -> str:
>         """Check if the user has approved the plan or provided feedback"""
>         if state["user_feedback"] == "APPROVED":
>             return "approved"
>         return "feedback"
>     
>     def _check_sections_status(self, state: DeepReportState) -> str:
>         """Check if all sections are completed"""
>         # Check if all main sections are done
>         all_sections_done = all(
>             section.status == "completed" 
>             for section in state["report_plan"].sections
>         )
>         
>         if all_sections_done and not state["report_plan"].introduction:
>             return "introduction"
>         elif all_sections_done:
>             return "all_done"
>         else:
>             return "next_section"
>     
>     async def _plan_report(self, state: DeepReportState) -> Dict[str, Any]:
>         """Create a plan for the report"""
>         config = state["config"]
>         query = state["query"]
>         
>         # Get the planner LLM
>         llm = get_llm(
>             provider=config.get("planner_provider", "openai"),
>             model=config.get("planner_model", "gpt-4o")
>         )
>         
>         # Create the planning prompt
>         prompt = f"""
>         You are a research expert planning a detailed report on the following topic:
>         
>         {query}
>         
>         Based on this topic, create a structured research report plan with 3-7 main sections.
>         For each section, provide a clear title that indicates what will be covered.
>         """
>         
>         # If there's a custom report structure, include it
>         if "report_structure" in config:
>             prompt += f"\n\nUse the following structure for your report:\n{config['report_structure']}"
>         
>         # If there's user feedback, include it
>         if state.get("user_feedback") and state["user_feedback"] != "APPROVED":
>             prompt += f"\n\nConsider this feedback when revising the plan:\n{state['user_feedback']}"
>         
>         # Generate the report plan
>         sections_schema = {
>             "title": "Report title",
>             "sections": [
>                 {
>                     "title": "Section title",
>                 }
>             ]
>         }
>         
>         plan_response = await llm.astructured_generate(
>             prompt, 
>             response_schema=sections_schema,
>             temperature=0.7
>         )
>         
>         # Convert to ReportPlan object
>         report_plan = ReportPlan(
>             title=plan_response["title"],
>             sections=[ReportSection(**section) for section in plan_response["sections"]]
>         )
>         
>         return {
>             "report_plan": report_plan,
>             "awaiting_user_feedback": True,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "report_plan", "plan": report_plan}
>             ]
>         }
>     
>     async def _wait_for_user_feedback(self, state: DeepReportState) -> Dict[str, Any]:
>         """Wait for user feedback - this is a placeholder that would be triggered by API calls"""
>         # In a real implementation, this would wait for an external event
>         # For now, we'll simulate a hardcoded approval
>         return {"user_feedback": state.get("user_feedback", "APPROVED")}
>     
>     async def _process_user_feedback(self, state: DeepReportState) -> Dict[str, Any]:
>         """Process user feedback on the report plan"""
>         # Nothing to do here, as the feedback is passed back to plan_report
>         return {
>             "awaiting_user_feedback": False,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "user_feedback", "feedback": state["user_feedback"]}
>             ]
>         }
>     
>     async def _select_next_section(self, state: DeepReportState) -> Dict[str, Any]:
>         """Select the next section to work on"""
>         report_plan = state["report_plan"]
>         
>         # Find the first section that's not completed
>         for i, section in enumerate(report_plan.sections):
>             if section.status in ["planned", "researching"]:
>                 return {"current_section_index": i}
>         
>         # If all sections are done, return None
>         return {"current_section_index": None}
>     
>     async def _research_section(self, state: DeepReportState) -> Dict[str, Any]:
>         """Research a specific section"""
>         config = state["config"]
>         section_index = state["current_section_index"]
>         section = state["report_plan"].sections[section_index]
>         
>         # Get the researcher LLM
>         llm = get_llm(
>             provider=config.get("researcher_provider", "openai"),
>             model=config.get("researcher_model", "gpt-4o")
>         )
>         
>         # Create the research planning prompt
>         prompt = f"""
>         You are researching the following section of a report:
>         
>         Report Title: {state["report_plan"].title}
>         Section Title: {section.title}
>         
>         Generate {config.get('queries_per_section', 3)} specific search queries that will help gather 
>         comprehensive information for this section.
>         """
>         
>         # Generate search queries
>         queries_schema = {
>             "queries": ["Search query 1", "Search query 2", "Search query 3"]
>         }
>         
>         query_response = await llm.astructured_generate(
>             prompt, 
>             response_schema=queries_schema,
>             temperature=0.7
>         )
>         
>         # Execute searches
>         search_provider = get_search_provider(
>             provider=config.get("search_provider", "tavily"),
>             api_key=config.get(f"{config.get('search_provider', 'tavily').upper()}_API_KEY")
>         )
>         
>         all_results = []
>         for query in query_response["queries"]:
>             results = await search_provider.asearch(query)
>             all_results.append({"query": query, "results": results})
>         
>         # Update the section in the report plan
>         updated_sections = state["report_plan"].sections.copy()
>         updated_sections[section_index].status = "writing"
>         updated_sections[section_index].research_queries = query_response["queries"]
>         updated_sections[section_index].research_results = all_results
>         
>         updated_report_plan = state["report_plan"].copy()
>         updated_report_plan.sections = updated_sections
>         
>         return {
>             "report_plan": updated_report_plan,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "section_research", "section": section.title, "results": all_results}
>             ]
>         }
>     
>     async def _write_section(self, state: DeepReportState) -> Dict[str, Any]:
>         """Write a specific section"""
>         config = state["config"]
>         section_index = state["current_section_index"]
>         section = state["report_plan"].sections[section_index]
>         
>         # Get the writer LLM
>         llm = get_llm(
>             provider=config.get("writer_provider", "anthropic"),
>             model=config.get("writer_model", "claude-3-opus")
>         )
>         
>         # Create the writing prompt
>         prompt = f"""
>         You are writing a section of a research report. Use the research results to write a comprehensive, 
>         well-structured section with proper citations.
>         
>         Report Title: {state["report_plan"].title}
>         Section Title: {section.title}
>         
>         Research Results:
>         """
>         
>         for i, research in enumerate(section.research_results):
>             prompt += f"\nQuery: {research['query']}\n"
>             for j, result in enumerate(research["results"]):
>                 prompt += f"- Source {i+1}.{j+1}: {result['title']}\n  URL: {result['url']}\n  Snippet: {result['snippet']}\n"
>         
>         prompt += "\nWrite a comprehensive section with proper citations in markdown format."
>         
>         # Generate the section content
>         section_content = await llm.agenerate(prompt, temperature=0.7)
>         
>         # Update the section in the report plan
>         updated_sections = state["report_plan"].sections.copy()
>         updated_sections[section_index].status = "completed"
>         updated_sections[section_index].content = section_content
>         
>         updated_report_plan = state["report_plan"].copy()
>         updated_report_plan.sections = updated_sections
>         
>         return {
>             "report_plan": updated_report_plan,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "section_content", "section": section.title, "content": section_content}
>             ]
>         }
>     
>     async def _write_introduction(self, state: DeepReportState) -> Dict[str, Any]:
>         """Write the introduction for the report"""
>         config = state["config"]
>         report_plan = state["report_plan"]
>         
>         # Get the writer LLM
>         llm = get_llm(
>             provider=config.get("writer_provider", "anthropic"),
>             model=config.get("writer_model", "claude-3-opus")
>         )
>         
>         # Create the introduction prompt
>         prompt = f"""
>         Write an engaging introduction for the following research report. The introduction should provide context, 
>         explain the importance of the topic, and preview the main sections.
>         
>         Report Title: {report_plan.title}
>         
>         Sections:
>         """
>         
>         for section in report_plan.sections:
>             prompt += f"- {section.title}\n"
>         
>         # Generate the introduction
>         introduction = await llm.agenerate(prompt, temperature=0.7)
>         
>         # Update the report plan
>         updated_report_plan = report_plan.copy()
>         updated_report_plan.introduction = introduction
>         
>         return {
>             "report_plan": updated_report_plan,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "introduction", "content": introduction}
>             ]
>         }
>     
>     async def _write_conclusion(self, state: DeepReportState) -> Dict[str, Any]:
>         """Write the conclusion for the report"""
>         config = state["config"]
>         report_plan = state["report_plan"]
>         
>         # Get the writer LLM
>         llm = get_llm(
>             provider=config.get("writer_provider", "anthropic"),
>             model=config.get("writer_model", "claude-3-opus")
>         )
>         
>         # Create the conclusion prompt
>         prompt = f"""
>         Write a thoughtful conclusion for the following research report. The conclusion should summarize key points, 
>         provide closure, and offer final thoughts or recommendations.
>         
>         Report Title: {report_plan.title}
>         
>         Sections Summary:
>         """
>         
>         for section in report_plan.sections:
>             # Include a brief summary of each section to inform the conclusion
>             prompt += f"- {section.title}: {section.content[:200]}...\n"
>         
>         # Generate the conclusion
>         conclusion = await llm.agenerate(prompt, temperature=0.7)
>         
>         # Update the report plan
>         updated_report_plan = report_plan.copy()
>         updated_report_plan.conclusion = conclusion
>         
>         return {
>             "report_plan": updated_report_plan,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "conclusion", "content": conclusion}
>             ]
>         }
>     
>     async def _finalize_report(self, state: DeepReportState) -> Dict[str, Any]:
>         """Finalize the report and create the full markdown document"""
>         report_plan = state["report_plan"]
>         
>         # Assemble the full report
>         full_report = f"# {report_plan.title}\n\n"
>         
>         if report_plan.introduction:
>             full_report += f"## Introduction\n\n{report_plan.introduction}\n\n"
>         
>         for section in report_plan.sections:
>             full_report += f"## {section.title}\n\n{section.content}\n\n"
>         
>         if report_plan.conclusion:
>             full_report += f"## Conclusion\n\n{report_plan.conclusion}\n\n"
>         
>         # Add references
>         full_report += "## References\n\n"
>         all_refs = []
>         for section in report_plan.sections:
>             for research in section.research_results:
>                 for result in research["results"]:
>                     all_refs.append(f"- [{result['title']}]({result['url']})")
>         
>         # Deduplicate references
>         unique_refs = list(set(all_refs))
>         full_report += "\n".join(unique_refs)
>         
>         return {
>             "final_answer": full_report,
>             "intermediate_results": state["intermediate_results"] + [
>                 {"type": "final_report", "content": full_report}
>             ]
>         }
> ```
> 
> 6. **app/services/llm/base.py**: Base LLM interface
> ```python
> from abc import ABC, abstractmethod
> from typing import Dict, Any, Optional, Union, Type
> from pydantic import BaseModel
> 
> class BaseLLM(ABC):
>     """Base class for all LLM interfaces"""
>     
>     def __init__(self, model: str, api_key: Optional[str] = None, **kwargs):
>         self.model = model
>         self.api_key = api_key
>         self.kwargs = kwargs
>     
>     @abstractmethod
>     async def agenerate(self, prompt: str, temperature: float = 0.7) -> str:
>         """Generate text from a prompt asynchronously"""
>         pass
>     
>     @abstractmethod
>     async def astructured_generate(
>         self, 
>         prompt: str, 
>         response_model: Optional[Union[Dict[str, Any], Type[BaseModel]]] = None,
>         temperature: float = 0.7
>     ) -> Dict[str, Any]:
>         """Generate structured output from a prompt asynchronously"""
>         pass
>     
>     @abstractmethod
>     async def astream_generate(self, prompt: str, temperature: float = 0.7):
>         """Stream generated text from a prompt asynchronously"""
>         pass
> ```
> 
> 7. **app/services/search/base.py**: Base search provider interface
> ```python
> from abc import ABC, abstractmethod
> from typing import Dict, Any, List, Optional
> 
> class BaseSearchProvider(ABC):
>     """Base class for all search providers"""
>     
>     def __init__(self, api_key: Optional[str] = None, **kwargs):
>         self.api_key = api_key
>         self.kwargs = kwargs
>     
>     @abstractmethod
>     async def asearch(self, query: str, num_results: int = 5) -> List[Dict[str, Any]]:
>         """Execute a search query asynchronously"""
>         pass
>     
>     @abstractmethod
>     async def asimilar(self, url: str, num_results: int = 5) -> List[Dict[str, Any]]:
>         """Find similar pages to a URL asynchronously"""
>         pass
> ```
> 
> 8. **app/api/endpoints/research.py**: Research endpoint
> ```python
> from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
> from fastapi.responses import StreamingResponse
> from app.models.schemas import ResearchRequest, ResearchResponse
> from app.graphs.quick_answer import QuickAnswerGraph
> from app.graphs.deep_report import DeepReportGraph
> from app.graphs.research import ResearchGraph
> from app.core.config import settings
> import uuid
> import asyncio
> from typing import Dict, Any, Optional
> 
> router = APIRouter()
> 
> active_sessions = {}
> 
> @router.post("/research", response_model=ResearchResponse)
> async def create_research(request: ResearchRequest, background_tasks: BackgroundTasks):
>     """Create a new research request"""
>     session_id = str(uuid.uuid4())
>     
>     # Initialize appropriate graph based on research type
>     if request.mode == "quick":
>         graph = QuickAnswerGraph(config=request.config)
>     elif request.mode == "report":
>         graph = DeepReportGraph(config=request.config)
>     else:  # Default to research mode
>         graph = ResearchGraph(config=request.config)
>     
>     # Store the graph and query in active sessions
>     active_sessions[session_id] = {
>         "graph": graph,
>         "query": request.query,
>         "status": "processing",
>         "result": None,
>         "events": [],
>     }
>     
>     # Start processing in background
>     background_tasks.add_task(process_research, session_id, request)
>     
>     return ResearchResponse(
>         session_id=session_id,
>         status="processing",
>         message="Research request started"
>     )
> 
> @router.get("/research/{session_id}", response_model=ResearchResponse)
> async def get_research_status(session_id: str):
>     """Check the status of a research request"""
>     if session_id not in active_sessions:
>         raise HTTPException(status_code=404, detail="Research session not found")
>     
>     session = active_sessions[session_id]
>     
>     return ResearchResponse(
>         session_id=session_id,
>         status=session["status"],
>         message=f"Research is {session['status']}",
>         result=session.get("result"),
>     )
> 
> @router.get("/research/{session_id}/stream")
> async def stream_research_events(session_id: str):
>     """Stream events from a research session"""
>     if session_id not in active_sessions:
>         raise HTTPException(status_code=404, detail="Research session not found")
>     
>     # Create an async generator to stream events
>     async def event_generator():
>         session = active_sessions[session_id]
>         last_event_index = 0
>         
>         while session["status"] == "processing":
>             if len(session["events"]) > last_event_index:
>                 # Send all new events
>                 for event in session["events"][last_event_index:]:
>                     yield f"data: {event}\n\n"
>                 last_event_index = len(session["events"])
>             
>             # Wait a bit before checking again
>             await asyncio.sleep(0.5)
>         
>         # Send final event
>         yield f"data: {{'event': 'complete', 'result': {session.get('result', '{}')}}\n\n"
>     
>     return StreamingResponse(
>         event_generator(),
>         media_type="text/event-stream"
>     )
> 
> @router.post("/research/{session_id}/feedback")
> async def provide_feedback(session_id: str, feedback: Optional[str] = None):
>     """Provide feedback for a research plan"""
>     if session_id not in active_sessions:
>         raise HTTPException(status_code=404, detail="Research session not found")
>     
>     session = active_sessions[session_id]
>     
>     # Store the feedback in the session
>     session["user_feedback"] = feedback if feedback else "APPROVED"
>     
>     return {"status": "feedback_received"}
> 
> async def process_research(session_id: str, request: ResearchRequest):
>     """Process a research request in the background"""
>     session = active_sessions[session_id]
>     
>     try:
>         # Run the graph asynchronously and collect events
>         async for event in session["graph"].astream(request.query, request.config):
>             session["events"].append(event)
>         
>         # Get the final result
>         result = await session["graph"].arun(request.query, request.config)
>         
>         # Update session with the result
>         session["status"] = "completed"
>         session["result"] = result.get("final_answer")
>         
>     except Exception as e:
>         # Handle errors
>         session["status"] = "error"
>         session["error"] = str(e)
> ```
> 
> #### Frontend:
> 
> 1. **src/App.jsx**: Main React application
> ```jsx
> import React from 'react';
> import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
> import { ThemeProvider } from './contexts/ThemeContext';
> import { AuthProvider } from './contexts/AuthContext';
> import { ResearchProvider } from './contexts/ResearchContext';
> import Header from './components/common/Header';
> import Footer from './components/common/Footer';
> import HomePage from './pages/HomePage';
> import QuickAnswerPage from './pages/QuickAnswerPage';
> import DeepReportPage from './pages/DeepReportPage';
> import ResearchPage from './pages/ResearchPage';
> import SettingsPage from './pages/SettingsPage';
> import NotFoundPage from './pages/NotFoundPage';
> 
> function App() {
>   return (
>     <ThemeProvider>
>       <AuthProvider>
>         <ResearchProvider>
>           <Router>
>             <div className="flex flex-col min-h-screen">
>               <Header />
>               <main className="flex-grow container mx-auto px-4 py-6">
>                 <Routes>
>                   <Route path="/" element={<HomePage />} />
>                   <Route path="/quick-answer" element={<QuickAnswerPage />} />
>                   <Route path="/deep-report" element={<DeepReportPage />} />
>                   <Route path="/research" element={<ResearchPage />} />
>                   <Route path="/settings" element={<SettingsPage />} />
>                   <Route path="*" element={<NotFoundPage />} />
>                 </Routes>
>               </main>
>               <Footer />
>             </div>
>           </Router>
>         </ResearchProvider>
>       </AuthProvider>
>     </ThemeProvider>
>   );
> }
> 
> export default App;
> ```
> 
> 2. **src/components/Research/ResearchInput.jsx**: Research input component
> ```jsx
> import React, { useState } from 'react';
> import { useResearch } from '../../contexts/ResearchContext';
> import Button from '../common/Button';
> import Select from '../common/Select';
> import TextField from '../common/TextField';
> 
> const ResearchInput = () => {
>   const { startResearch, isLoading } = useResearch();
>   const [query, setQuery] = useState('');
>   const [mode, setMode] = useState('quick');
>   const [showAdvanced, setShowAdvanced] = useState(false);
>   const [config, setConfig] = useState({
>     search_provider: 'tavily',
>     planner_provider: 'openai',
>     planner_model: 'gpt-4o',
>     writer_provider: 'anthropic',
>     writer_model: 'claude-3-opus',
>     max_search_depth: 2,
>     queries_per_section: 3
>   });
> 
>   const handleInputChange = (e) => {
>     setQuery(e.target.value);
>   };
> 
>   const handleModeChange = (e) => {
>     setMode(e.target.value);
>   };
> 
>   const handleConfigChange = (e) => {
>     const { name, value } = e.target;
>     setConfig(prev => ({
>       ...prev,
>       [name]: name.includes('max_') || name.includes('_per_') 
>         ? parseInt(value, 10) 
>         : value
>     }));
>   };
> 
>   const handleSubmit = (e) => {
>     e.preventDefault();
>     if (query.trim()) {
>       startResearch(query, mode, config);
>     }
>   };
> 
>   return (
>     <div className="bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 mb-6">
>       <form onSubmit={handleSubmit}>
>         <div className="mb-4">
>           <TextField
>             label="What would you like to research?"
>             value={query}
>             onChange={handleInputChange}
>             placeholder="Enter your research query..."
>             fullWidth
>             required
>             multiline
>             minRows={2}
>           />
>         </div>
>         
>         <div className="mb-4">
>           <Select
>             label="Research Mode"
>             value={mode}
>             onChange={handleModeChange}
>             options={[
>               { value: 'quick', label: 'Quick Answer - Find answers fast' },
>               { value: 'report', label: 'Deep Report - Comprehensive research report' },
>               { value: 'research', label: 'Research - Systematic investigation' }
>             ]}
>           />
>         </div>
>         
>         <div className="flex justify-between items-center mb-4">
>           <Button
>             type="button"
>             variant="text"
>             onClick={() => setShowAdvanced(!showAdvanced)}
>           >
>             {showAdvanced ? 'Hide Advanced Options' : 'Show Advanced Options'}
>           </Button>
>           
>           <Button
>             type="submit"
>             variant="contained"
>             color="primary"
>             disabled={isLoading || !query.trim()}
>           >
>             {isLoading ? 'Researching...' : 'Start Research'}
>           </Button>
>         </div>
>         
>         {showAdvanced && (
>           <div className="grid grid-cols-1 md:grid-cols-2 gap-4 p-4 bg-gray-50 dark:bg-gray-700 rounded-lg">
>             <Select
>               name="search_provider"
>               label="Search Provider"
>               value={config.search_provider}
>               onChange={handleConfigChange}
>               options={[
>                 { value: 'tavily', label: 'Tavily' },
>                 { value: 'perplexity', label: 'Perplexity' },
>                 { value: 'exa', label: 'Exa' },
>                 { value: 'arxiv', label: 'ArXiv' },
>                 { value: 'pubmed', label: 'PubMed' },
>               ]}
>             />
>             
>             <Select
>               name="planner_provider"
>               label="Planner Model Provider"
>               value={config.planner_provider}
>               onChange={handleConfigChange}
>               options={[
>                 { value: 'openai', label: 'OpenAI' },
>                 { value: 'anthropic', label: 'Anthropic' },
>                 { value: 'cohere', label: 'Cohere' },
>               ]}
>             />
>             
>             <TextField
>               name="planner_model"
>               label="Planner Model"
>               value={config.planner_model}
>               onChange={handleConfigChange}
>               placeholder="e.g., gpt-4o"
>             />
>             
>             <Select
>               name="writer_provider"
>               label="Writer Model Provider"
>               value={config.writer_provider}
>               onChange={handleConfigChange}
>               options={[
>                 { value: 'openai', label: 'OpenAI' },
>                 { value: 'anthropic', label: 'Anthropic' },
>                 { value: 'cohere', label: 'Cohere' },
>               ]}
>             />
>             
>             <TextField
>               name="writer_model"
>               label="Writer Model"
>               value={config.writer_model}
>               onChange={handleConfigChange}
>               placeholder="e.g., claude-3-opus"
>             />
>             
>             <TextField
>               name="max_search_depth"
>               label="Max Search Depth"
>               type="number"
>               value={config.max_search_depth}
>               onChange={handleConfigChange}
>               min={1}
>               max={5}
>             />
>             
>             <TextField
>               name="queries_per_section"
>               label="Queries Per Section"
>               type="number"
>               value={config.queries_per_section}
>               onChange={handleConfigChange}
>               min={1}
>               max={10}
>             />
>           </div>
>         )}
>       </form>
>     </div>
>   );
> };
> 
> export default ResearchInput;
> ```
> 
> 3. **src/components/Research/ResearchProcessVisualization.jsx**: Research process visualization
> ```jsx
> import React from 'react';
> import { useResearch } from '../../contexts/ResearchContext';
> import { FaSearch, FaBook, FaBrain, FaFileAlt } from 'react-icons/fa';
> 
> const ResearchProcessVisualization = () => {
>   const { events, status } = useResearch();
>   
>   // Group events by type for visualization
>   const searchEvents = events.filter(e => e.type === 'search');
>   const visitEvents = events.filter(e => e.type === 'visit');
>   const thinkingEvents = events.filter(e => e.type === 'thinking');
>   const planEvents = events.filter(e => e.type === 'report_plan');
>   
>   // Calculate stats
>   const totalSearches = searchEvents.length;
>   const totalVisits = visitEvents.reduce((sum, e) => sum + (e.urls?.length || 0), 0);
>   const totalThinking = thinkingEvents.length;
>   
>   return (
>     <div className="bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 mb-6">
>       <h2 className="text-xl font-semibold mb-4">Research Process</h2>
>       
>       <div className="flex flex-wrap gap-4 mb-6">
>         <div className="flex-1 bg-blue-50 dark:bg-blue-900 p-4 rounded-lg">
>           <div className="flex items-center mb-2">
>             <FaSearch className="text-blue-500 mr-2" />
>             <h3 className="font-medium">Searches</h3>
>           </div>
>           <p className="text-2xl font-bold">{totalSearches}</p>
>         </div>
>         
>         <div className="flex-1 bg-green-50 dark:bg-green-900 p-4 rounded-lg">
>           <div className="flex items-center mb-2">
>             <FaBook className="text-green-500 mr-2" />
>             <h3 className="font-medium">Pages Visited</h3>
>           </div>
>           <p className="text-2xl font-bold">{totalVisits}</p>
>         </div>
>         
>         <div className="flex-1 bg-purple-50 dark:bg-purple-900 p-4 rounded-lg">
>           <div className="flex items-center mb-2">
>             <FaBrain className="text-purple-500 mr-2" />
>             <h3 className="font-medium">Thinking Steps</h3>
>           </div>
>           <p className="text-2xl font-bold">{totalThinking}</p>
>         </div>
>         
>         <div className="flex-1 bg-amber-50 dark:bg-amber-900 p-4 rounded-lg">
>           <div className="flex items-center mb-2">
>             <FaFileAlt className="text-amber-500 mr-2" />
>             <h3 className="font-medium">Status</h3>
>           </div>
>           <p className="text-lg font-medium capitalize">{status}</p>
>         </div>
>       </div>
>       
>       {/* Timeline visualization */}
>       <div className="relative">
>         <div className="absolute left-4 top-0 bottom-0 w-0.5 bg-gray-200 dark:bg-gray-600"></div>
>         
>         {events.map((event, index) => (
>           <div key={index} className="ml-10 mb-4 relative">
>             <div className="absolute -left-10 mt-1.5">
>               {event.type === 'search' && <FaSearch className="text-blue-500" />}
>               {event.type === 'visit' && <FaBook className="text-green-500" />}
>               {event.type === 'thinking' && <FaBrain className="text-purple-500" />}
>               {event.type === 'report_plan' && <FaFileAlt className="text-amber-500" />}
>             </div>
>             
>             <div className="bg-gray-50 dark:bg-gray-700 p-3 rounded-lg">
>               <p className="font-medium capitalize">{event.type}</p>
>               {event.type === 'search' && (
>                 <p className="text-sm text-gray-600 dark:text-gray-300">
>                   Query: {event.query}
>                 </p>
>               )}
>               {event.type === 'visit' && (
>                 <div>
>                   <p className="text-sm text-gray-600 dark:text-gray-300">
>                     Visited {event.urls?.length || 0} pages
>                   </p>
>                   <ul className="text-xs text-gray-500 dark:text-gray-400 mt-1">
>                     {event.urls?.map((url, i) => (
>                       <li key={i} className="truncate">{url}</li>
>                     ))}
>                   </ul>
>                 </div>
>               )}
>             </div>
>           </div>
>         ))}
>       </div>
>     </div>
>   );
> };
> 
> export default ResearchProcessVisualization;
> ```
> 
> 4. **src/contexts/ResearchContext.jsx**: Research context provider
> ```jsx
> import React, { createContext, useContext, useState, useRef, useEffect } from 'react';
> import { researchApi } from '../services/api';
> 
> const ResearchContext = createContext();
> 
> export const useResearch = () => useContext(ResearchContext);
> 
> export const ResearchProvider = ({ children }) => {
>   const [query, setQuery] = useState('');
>   const [mode, setMode] = useState('');
>   const [config, setConfig] = useState({});
>   const [sessionId, setSessionId] = useState(null);
>   const [status, setStatus] = useState('idle');
>   const [result, setResult] = useState(null);
>   const [events, setEvents] = useState([]);
>   const [isLoading, setIsLoading] = useState(false);
>   const [error, setError] = useState(null);
>   const eventSourceRef = useRef(null);
> 
>   const startResearch = async (queryText, modeType, configOptions = {}) => {
>     try {
>       setIsLoading(true);
>       setQuery(queryText);
>       setMode(modeType);
>       setConfig(configOptions);
>       setEvents([]);
>       setResult(null);
>       setError(null);
>       setStatus('starting');
>       
>       // Start a new research session
>       const response = await researchApi.createResearch({
>         query: queryText,
>         mode: modeType,
>         config: configOptions
>       });
>       
>       setSessionId(response.session_id);
>       setStatus('processing');
>       
>       // Connect to the event stream
>       connectToEventStream(response.session_id);
>       
>     } catch (err) {
>       setError(err.message || 'Failed to start research');
>       setStatus('error');
>       setIsLoading(false);
>     }
>   };
>   
>   const connectToEventStream = (id) => {
>     // Close existing connection if any
>     if (eventSourceRef.current) {
>       eventSourceRef.current.close();
>     }
>     
>     // Connect to the server-sent events endpoint
>     const eventSource = new EventSource(`/api/v1/research/${id}/stream`);
>     eventSourceRef.current = eventSource;
>     
>     eventSource.onmessage = (event) => {
>       const data = JSON.parse(event.data);
>       
>       if (data.event === 'complete') {
>         // Research is complete
>         setResult(data.result);
>         setStatus('completed');
>         setIsLoading(false);
>         eventSource.close();
>       } else {
>         // Add event to the list
>         setEvents(prev => [...prev, data]);
>       }
>     };
>     
>     eventSource.onerror = () => {
>       eventSource.close();
>       setError('Connection to server lost');
>       // Try to recover the state
>       checkResearchStatus(id);
>     };
>   };
>   
>   const checkResearchStatus = async (id) => {
>     try {
>       const response = await researchApi.getResearchStatus(id);
>       
>       setStatus(response.status);
>       
>       if (response.status === 'completed' || response.status === 'error') {
>         setResult(response.result);
>         setIsLoading(false);
>       } else if (response.status === 'processing') {
>         // Try to reconnect to the event stream
>         setTimeout(() => connectToEventStream(id), 1000);
>       }
>     } catch (err) {
>       setError(err.message || 'Failed to check research status');
>       setStatus('error');
>       setIsLoading(false);
>     }
>   };
> 
>   const provideFeedback = async (feedback) => {
>     if (!sessionId) return;
>     
>     try {
>       await researchApi.provideFeedback(sessionId, feedback);
>       // The server will continue processing after feedback
>     } catch (err) {
>       setError(err.message || 'Failed to provide feedback');
>     }
>   };
>   
>   // Clean up event source on unmount
>   useEffect(() => {
>     return () => {
>       if (eventSourceRef.current) {
>         eventSourceRef.current.close();
>       }
>     };
>   }, []);
> 
>   return (
>     <ResearchContext.Provider value={{
>       query,
>       mode,
>       config,
>       sessionId,
>       status,
>       result,
>       events,
>       isLoading,
>       error,
>       startResearch,
>       provideFeedback
>     }}>
>       {children}
>     </ResearchContext.Provider>
>   );
> };
> ```
> 
> 5. **src/pages/DeepReportPage.jsx**: Deep report page
> ```jsx
> import React, { useState } from 'react';
> import { useResearch } from '../contexts/ResearchContext';
> import ResearchInput from '../components/Research/ResearchInput';
> import ResearchProcessVisualization from '../components/Research/ResearchProcessVisualization';
> import ReportPlanReview from '../components/Report/ReportPlanReview';
> import ReportViewer from '../components/Report/ReportViewer';
> import Button from '../components/common/Button';
> import Alert from '../components/common/Alert';
> import Loader from '../components/common/Loader';
> import { FaDownload, FaShareSquare } from 'react-icons/fa';
> 
> const DeepReportPage = () => {
>   const { 
>     status, 
>     result, 
>     events, 
>     isLoading, 
>     error, 
>     provideFeedback 
>   } = useResearch();
>   
>   const [exportFormat, setExportFormat] = useState('markdown');
>   
>   // Find the report plan event if it exists
>   const reportPlanEvent = events.find(e => e.type === 'report_plan');
>   
>   const handleApproveReportPlan = () => {
>     provideFeedback('APPROVED');
>   };
>   
>   const handleProvideReportFeedback = (feedback) => {
>     provideFeedback(feedback);
>   };
>   
>   const handleExportReport = () => {
>     if (!result) return;
>     
>     let exportContent = result;
>     let mimeType = 'text/markdown';
>     let extension = 'md';
>     
>     if (exportFormat === 'pdf') {
>       // This is a placeholder - in a real implementation,
>       // you would convert markdown to PDF using a library
>       mimeType = 'application/pdf';
>       extension = 'pdf';
>       // exportContent = convertToPdf(result);
>     } else if (exportFormat === 'html') {
>       // Convert markdown to HTML
>       mimeType = 'text/html';
>       extension = 'html';
>       // exportContent = markdownToHtml(result);
>     }
>     
>     // Create a blob and trigger download
>     const blob = new Blob([exportContent], { type: mimeType });
>     const url = URL.createObjectURL(blob);
>     const a = document.createElement('a');
>     a.href = url;
>     a.download = `report.${extension}`;
>     document.body.appendChild(a);
>     a.click();
>     document.body.removeChild(a);
>     URL.revokeObjectURL(url);
>   };
>   
>   const handleShareReport = () => {
>     // In a real implementation, this would open a share dialog
>     // or create a shareable link
>     alert('Share functionality would be implemented here');
>   };
>   
>   return (
>     <div>
>       <div className="mb-8">
>         <h1 className="text-3xl font-bold mb-2">Deep Research Report</h1>
>         <p className="text-gray-600 dark:text-gray-300">
>           Create comprehensive, well-structured research reports on any topic.
>         </p>
>       </div>
>       
>       {error && (
>         <Alert 
>           type="error" 
>           title="Error" 
>           message={error}
>           className="mb-6"
>         />
>       )}
>       
>       <ResearchInput />
>       
>       {isLoading && (
>         <div className="flex justify-center my-12">
>           <Loader size="large" />
>         </div>
>       )}
>       
>       {status !== 'idle' && !isLoading && (
>         <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
>           <div className="lg:col-span-1">
>             <ResearchProcessVisualization />
>           </div>
>           
>           <div className="lg:col-span-2">
>             {reportPlanEvent && status === 'awaiting_feedback' && (
>               <ReportPlanReview 
>                 plan={reportPlanEvent.plan}
>                 onApprove={handleApproveReportPlan}
>                 onFeedback={handleProvideReportFeedback}
>               />
>             )}
>             
>             {result && (
>               <>
>                 <div className="bg-white dark:bg-gray-800 rounded-lg shadow-md p-6 mb-6">
>                   <div className="flex justify-between items-center mb-4">
>                     <h2 className="text-xl font-semibold">Final Report</h2>
>                     <div className="flex space-x-2">
>                       <select
>                         value={exportFormat}
>                         onChange={(e) => setExportFormat(e.target.value)}
>                         className="bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded px-3 py-1 text-sm"
>                       >
>                         <option value="markdown">Markdown</option>
>                         <option value="html">HTML</option>
>                         <option value="pdf">PDF</option>
>                       </select>
>                       <Button
>                         onClick={handleExportReport}
>                         variant="outlined"
>                         size="small"
>                         startIcon={<FaDownload />}
>                       >
>                         Export
>                       </Button>
>                       <Button
>                         onClick={handleShareReport}
>                         variant="outlined"
>                         size="small"
>                         startIcon={<FaShareSquare />}
>                       >
>                         Share
>                       </Button>
>                     </div>
>                   </div>
>                   
>                   <ReportViewer report={result} />
>                 </div>
>               </>
>             )}
>           </div>
>         </div>
>       )}
>     </div>
>   );
> };
> 
> export default DeepReportPage;
> ```
> 
> ### Trade-offs and Considerations:
> 
> 1. **Workflow Approach**:
>    - **LangGraph vs Custom**: LangGraph provides a structured way to define complex workflows, but may have overhead for simpler tasks.
>    - **Monolithic vs Modular**: The design allows for multiple workflow types (quick answer, deep report, research) to be selected based on user needs.
> 
> 2. **Model Usage**:
>    - **Multiple Models vs Single Model**: Using different models for planning vs writing allows optimizing for different strengths.
>    - **Context Management**: Managing context size across multiple API calls to avoid exceeding limits.
> 
> 3. **Search Providers**:
>    - **Multiple Providers vs Single Provider**: Supporting multiple search APIs increases development complexity but offers more flexibility.
>    - **Domain-Specific Search**: Including specialized search tools like ArXiv and PubMed enhances domain-specific research.
> 
> 4. **UI Complexity**:
>    - **Real-time Updates**: Streaming events from the backend increases implementation complexity but provides a better user experience.
>    - **Visualization vs Simplicity**: The design includes process visualization which adds complexity but improves transparency.
> 
> 5. **Deployment Considerations**:
>    - **API Key Management**: The system requires managing multiple API keys securely.
>    - **Scalability**: The backend needs to handle long-running processes and manage state across requests.
> 
> ### Implementation Plan:
> 
> 1. **Phase 1: Core Backend**
>    - Implement base LangGraph classes and workflow definitions
>    - Set up LLM service interfaces
>    - Implement search provider interfaces
>    - Create basic API endpoints
> 
> 2. **Phase 2: Frontend Skeleton**
>    - Set up React application structure
>    - Create core components and contexts
>    - Implement basic styling
> 
> 3. **Phase 3: Integration**
>    - Connect frontend and backend
>    - Implement event streaming
>    - Add visualization components
> 
> 4. **Phase 4: Refinement and Testing**
>    - Optimize prompts and workflows
>    - Add error handling and recovery
>    - Performance testing and optimization
> 
> 5. **Phase 5: Advanced Features**
>    - Add report export functionality
>    - Implement sharing capabilities
>    - Add user preferences and history
> 
> This combined implementation takes the best aspects of each approach:
> 1. From SmolaAgents: Benchmarking approach and focus on performance
> 2. From Jina AI: The iterative search-read-reason loop and token budget management
> 3. From LangChain: Customizable report generation and human-in-the-loop feedback
> 
> The result is a flexible, powerful deep research system that can adapt to different user needs and research types.


# Comparison of Deep Search Implementations

## Similarities

1. **Common Goal**: All three implementations aim to replicate advanced search capabilities similar to commercial tools like OpenAI's Deep Research, Perplexity, and Grok.

2. **External Search Integration**: All use external search APIs to gather information from the web.

3. **LLM-powered Reasoning**: Each implementation utilizes large language models to analyze search results and synthesize answers.

4. **API-based Architecture**: All require API keys for various services (search providers and LLMs).

## Key Differences

### Hugging Face's SmolaAgents

- **Focus**: Benchmark-oriented implementation aiming to match OpenAI's Deep Research performance
- **Performance**: Achieves 55% pass@1 on the GAIA validation set (vs 67% for original Deep Research)
- **Language**: Python
- **Search Tool**: Primarily GoogleSearchTool with different providers
- **Interface**: Simple CLI-based interface
- **Architecture**: Relatively straightforward implementation

### Jina AI's node-DeepResearch

- **Focus**: Finding concise, accurate answers via an iterative process
- **Architecture**: Implements a clear Search → Read → Reason loop until token budget is reached
- **Language**: TypeScript/Node.js
- **Search**: Uses Jina Reader for searching and page content extraction
- **Models**: Supports Gemini, OpenAI, and LocalLLM options
- **Interface**: Provides both UI visualization and OpenAI-compatible API
- **Deployment**: Offers Docker setup and hosted service options
- **Optimization**: Explicitly states it doesn't optimize for long-form content (unlike OpenAI/Perplexity)

### LangChain's Open Deep Research

- **Focus**: Creating customizable research reports with structured sections
- **Architecture**: Plan-and-execute workflow with separate planning and research phases
- **Framework**: Built on LangGraph for workflow orchestration
- **Language**: Python
- **Search Tools**: Wide range of tools (Tavily, Perplexity, Exa, ArXiv, PubMed, etc.)
- **Models**: Uses separate planner and writer models
- **Human Interaction**: Supports human-in-the-loop approval of report plans
- **Customization**: Highly configurable (report structure, search depth, model selection)
- **UI**: Uses LangGraph Studio for visualization and interaction

## Best Combined Implementation Using LangGraph

### Architecture Design

```
DeepResearch/
├── backend/
│   ├── app/
│   │   ├── api/            # FastAPI endpoints
│   │   ├── core/           # Core configuration
│   │   ├── graphs/         # LangGraph workflows
│   │   ├── models/         # Data models
│   │   ├── services/       # LLM and search services
│   │   └── utils/          # Helper functions
│   └── tests/              # Backend tests
└── frontend/
    ├── public/
    ├── src/
    │   ├── components/     # React components
    │   ├── contexts/       # React contexts
    │   ├── pages/          # Page components
    │   └── services/       # API clients
    └── package.json
```

### Core Components

1. **Workflow Modes**
   - **Quick Answer Mode**: Inspired by Jina AI's approach, focused on finding concise answers efficiently
   - **Deep Report Mode**: Based on LangChain's approach, creating comprehensive research reports
   - **Research Mode**: A hybrid approach for systematic investigation with deeper reasoning

2. **LangGraph Workflows**

```python
# Simplified example
class BaseGraph:
    def __init__(self, config):
        self.config = config
        self.graph = self._build_graph()
    
    def _build_graph(self):
        raise NotImplementedError
    
    async def arun(self, query, config=None):
        # Implementation
        pass

class QuickAnswerGraph(BaseGraph):
    def _build_graph(self):
        workflow = StateGraph(QuickAnswerState)
        
        # Define nodes for search-read-reason loop (Jina approach)
        workflow.add_node("decide_next_action", self._decide_next_action)
        workflow.add_node("perform_search", self._perform_search)
        workflow.add_node("visit_url", self._visit_url)
        workflow.add_node("think", self._think)
        workflow.add_node("generate_answer", self._generate_answer)
        
        # Define edges and conditions
        # ...
        
        return workflow

class DeepReportGraph(BaseGraph):
    def _build_graph(self):
        workflow = StateGraph(DeepReportState)
        
        # Define nodes for plan-and-execute approach (LangChain style)
        workflow.add_node("plan_report", self._plan_report)
        workflow.add_node("wait_for_user_feedback", self._wait_for_user_feedback)
        workflow.add_node("research_section", self._research_section)
        workflow.add_node("write_section", self._write_section)
        # ...
        
        return workflow
```

3. **Unified Search Interface**

```python
class BaseSearchProvider:
    async def asearch(self, query, num_results=5):
        raise NotImplementedError
    
# Implementations for different search providers
class TavilySearch(BaseSearchProvider):
    # Implementation

class PerplexitySearch(BaseSearchProvider):
    # Implementation

class ArXivSearch(BaseSearchProvider):
    # Implementation
```

4. **Configurable LLM Service**

```python
class BaseLLM:
    async def agenerate(self, prompt, temperature=0.7):
        raise NotImplementedError
    
    async def astructured_generate(self, prompt, response_model=None, temperature=0.7):
        raise NotImplementedError
    
# LLM provider implementations
class OpenAIService(BaseLLM):
    # Implementation

class AnthropicService(BaseLLM):
    # Implementation
```

5. **React Frontend**

```jsx
// ResearchInput.jsx - Handle user queries and configuration
const ResearchInput = () => {
  const { startResearch } = useResearch();
  const [query, setQuery] = useState('');
  const [mode, setMode] = useState('quick');
  const [config, setConfig] = useState({/* defaults */});
  
  // Form handling and submission
  // ...
  
  return (
    <form onSubmit={handleSubmit}>
      <TextField
        label="What would you like to research?"
        value={query}
        onChange={handleInputChange}
      />
      <Select
        label="Research Mode"
        value={mode}
        onChange={handleModeChange}
        options={[
          { value: 'quick', label: 'Quick Answer' },
          { value: 'report', label: 'Deep Report' },
          { value: 'research', label: 'Research' }
        ]}
      />
      {/* Advanced options */}
    </form>
  );
};

// ProcessVisualization.jsx - Show the research process
const ResearchProcessVisualization = () => {
  const { events, status } = useResearch();
  
  // Process events for visualization
  // ...
  
  return (
    <div>
      {/* Statistics display */}
      {/* Timeline visualization */}
      {/* Current status */}
    </div>
  );
};
```

### Trade-offs and Design Considerations

1. **Workflow Complexity vs. Specialization**
   - Using separate graph implementations increases code complexity but allows for specialized handling of different research types
   - The design prioritizes flexibility over simplicity

2. **Model Selection Trade-offs**
   - Using different models for different tasks (planning, writing) optimizes for task-specific performance
   - Increases API costs but improves output quality

3. **Search Provider Trade-offs**
   - Supporting multiple search providers increases development complexity
   - Provides flexibility for different research domains (academic, news, general web)

4. **Human-in-the-loop vs. Automation**
   - The design includes human feedback capability for report planning
   - Adds UX complexity but improves result quality for complex tasks

5. **Performance Considerations**
   - Managing token budgets across multiple API calls
   - Parallel processing for report sections vs. sequential processing for quick answers

6. **Deployment Complexities**
   - Handling long-running processes
   - Secure management of multiple API keys
   - Streaming events from backend to frontend

### Implementation Strategy

1. Start with core LangGraph workflows focusing on the search-reason-read loop
2. Implement unified interfaces for search providers and LLM services
3. Build FastAPI backend with streaming support for real-time updates
4. Develop React frontend with components for different research modes
5. Add visualization components for research process transparency
6. Implement export and sharing capabilities for research results

This design combines the strengths of all three implementations:
- Jina AI's efficient search-read-reason loop
- LangChain's structured report generation with human feedback
- Hugging Face's focus on performance benchmarking

The result is a flexible deep research platform that can handle different research needs while maintaining transparency about the research process.
