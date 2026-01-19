# Data Integration Architecture: Multi-Source Financial Data Ingestion

> **Planning Document for bt-xy3: Other Data Shapes Integration**

## Executive Summary

This document specifies an extensible data ingestion architecture for Watchtower that extends beyond congressional disclosures to support multiple financial data types: insider trades (Form 4), institutional holdings (13F), earnings data, news, and social sentiment. The architecture provides a generic collector interface that enables rapid addition of new data sources while maintaining consistency, reliability, and validation standards.

## Context

### Current State
- **Watchtower**: Data ingestion system focused on congressional disclosures
- **Customs**: Schema/ontology layer with JSON Schema definitions
- **Bartertown**: Trading orchestration consuming validated data from Watchtower

### Design Principles (from existing architecture)
1. **Validation at Boundary**: All data validated against Customs schemas during ingestion
2. **Collector Interface Pattern**: Standardized collection approach
3. **Scheduled Orchestration**: Collectors run on schedules appropriate to data source
4. **Fault Tolerance**: Retry logic, deduplication, graceful degradation
5. **Attribution**: Every data point tracked to source and collection time

## Architecture Overview

### Data Flow

```
External Data Sources
    ↓ (collectors fetch)
Raw Data
    ↓ (parsers normalize)
Structured Data
    ↓ (validators check against Customs schemas)
Validated Data
    ↓ (writers persist)
Watchtower Data Store
    ↓ (consumers read)
Bartertown Trading System
```

### Component Layers

```
┌─────────────────────────────────────────────────────┐
│              Watchtower Orchestrator                │
│  (scheduling, monitoring, retry coordination)       │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│             Collector Interface Layer               │
│  (generic collector base, lifecycle management)    │
└─────────────────────────────────────────────────────┘
                        ↓
┌───────────┬──────────┬──────────┬─────────┬─────────┐
│Congressional│ Form 4 │   13F    │ Earnings│  News  │
│ Collector   │Collector│Collector │Collector│Collector│
└───────────┴──────────┴──────────┴─────────┴─────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                 Parser Layer                        │
│  (source-specific parsing, normalization)           │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              Validation Layer                       │
│  (validate against Customs JSON schemas)            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│               Storage Layer                         │
│  (persist to database, emit events)                 │
└─────────────────────────────────────────────────────┘
```

## Generic Collector Interface

### Base Collector Class

```python
# watchtower/collectors/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import Enum
from typing import Any, Dict, List, Optional
import structlog

log = structlog.get_logger()


class CollectorType(Enum):
    """Types of data collectors."""
    CONGRESSIONAL = "congressional"
    FORM_4 = "form_4"
    FORM_13F = "13f"
    EARNINGS = "earnings"
    NEWS = "news"
    SENTIMENT = "sentiment"


class DataFreshness(Enum):
    """How frequently data updates."""
    REAL_TIME = "real_time"        # < 1 minute
    NEAR_REAL_TIME = "near_real_time"  # 1-15 minutes
    HOURLY = "hourly"              # 1 hour
    DAILY = "daily"                # 1 day
    WEEKLY = "weekly"              # 1 week
    ON_DEMAND = "on_demand"        # Manual trigger


@dataclass
class CollectionConfig:
    """Configuration for a data collector."""
    collector_type: CollectorType
    enabled: bool
    schedule: str  # Cron expression
    timeout: int   # Seconds
    retry_attempts: int
    retry_backoff: float  # Exponential backoff multiplier
    batch_size: int
    lookback_days: int  # How far back to fetch on initial run

    # Rate limiting
    requests_per_minute: int
    requests_per_hour: int

    # Data source specific config
    source_config: Dict[str, Any]


@dataclass
class CollectionResult:
    """Result of a collection run."""
    collector_type: CollectorType
    run_id: str
    started_at: datetime
    completed_at: datetime
    status: str  # "success", "partial", "failed"

    # Counts
    records_fetched: int
    records_parsed: int
    records_validated: int
    records_stored: int
    records_duplicate: int
    records_failed: int

    # Error tracking
    errors: List[Dict[str, Any]]

    # Metadata
    metadata: Dict[str, Any]


class BaseCollector(ABC):
    """
    Base class for all data collectors.

    Lifecycle:
    1. fetch() - Retrieve raw data from external source
    2. parse() - Convert raw data to structured format
    3. validate() - Validate against Customs schemas
    4. deduplicate() - Remove duplicates
    5. store() - Persist to database
    6. emit_events() - Notify downstream consumers
    """

    def __init__(self, config: CollectionConfig):
        self.config = config
        self.log = log.bind(
            collector_type=config.collector_type.value
        )

    @abstractmethod
    def fetch(self, since: datetime) -> List[Any]:
        """
        Fetch raw data from external source.

        Args:
            since: Fetch data updated since this timestamp

        Returns:
            List of raw data objects (format depends on source)
        """
        pass

    @abstractmethod
    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """
        Parse raw data into structured format.

        Args:
            raw_data: Raw data from fetch()

        Returns:
            List of structured dictionaries matching Customs schema
        """
        pass

    @abstractmethod
    def get_schema_name(self) -> str:
        """
        Return the Customs schema name for validation.

        Returns:
            Schema name (e.g., "congressional_disclosure", "form_4")
        """
        pass

    def validate(self, structured_data: List[Dict[str, Any]]) -> tuple[List[Dict[str, Any]], List[Dict[str, Any]]]:
        """
        Validate structured data against Customs schema.

        Args:
            structured_data: Parsed data to validate

        Returns:
            Tuple of (valid_records, invalid_records)
        """
        from watchtower.validation.schema_validator import SchemaValidator

        validator = SchemaValidator(schema_name=self.get_schema_name())
        valid = []
        invalid = []

        for record in structured_data:
            result = validator.validate(record)
            if result.is_valid:
                valid.append(record)
            else:
                invalid.append({
                    "record": record,
                    "errors": result.errors
                })
                self.log.warning(
                    "validation_failed",
                    record_id=record.get("id"),
                    errors=result.errors
                )

        return valid, invalid

    def deduplicate(self, records: List[Dict[str, Any]]) -> tuple[List[Dict[str, Any]], int]:
        """
        Remove duplicate records.

        Args:
            records: Validated records

        Returns:
            Tuple of (unique_records, duplicate_count)
        """
        from watchtower.storage.deduplicator import Deduplicator

        dedup = Deduplicator(
            collector_type=self.config.collector_type
        )
        unique, duplicate_count = dedup.filter(records)

        self.log.info(
            "deduplication_complete",
            total=len(records),
            unique=len(unique),
            duplicates=duplicate_count
        )

        return unique, duplicate_count

    def store(self, records: List[Dict[str, Any]]) -> int:
        """
        Store validated records to database.

        Args:
            records: Validated, deduplicated records

        Returns:
            Number of records stored
        """
        from watchtower.storage.writer import DataWriter

        writer = DataWriter(
            collector_type=self.config.collector_type
        )
        stored = writer.write_batch(records)

        self.log.info(
            "storage_complete",
            records_stored=stored
        )

        return stored

    def emit_events(self, records: List[Dict[str, Any]]):
        """
        Emit events for new data.

        Args:
            records: Stored records to emit events for
        """
        from watchtower.events.emitter import EventEmitter

        emitter = EventEmitter()
        emitter.emit_new_data(
            collector_type=self.config.collector_type,
            records=records
        )

    def run(self, since: Optional[datetime] = None) -> CollectionResult:
        """
        Execute full collection pipeline.

        Args:
            since: Collect data since this timestamp (None = use config)

        Returns:
            CollectionResult with statistics and errors
        """
        import uuid
        from datetime import datetime, timedelta

        run_id = str(uuid.uuid4())
        started_at = datetime.now()

        if since is None:
            since = datetime.now() - timedelta(days=self.config.lookback_days)

        self.log.info(
            "collection_started",
            run_id=run_id,
            since=since.isoformat()
        )

        errors = []

        try:
            # 1. Fetch
            raw_data = self.fetch(since)
            records_fetched = len(raw_data)

            self.log.info("fetch_complete", count=records_fetched)

            # 2. Parse
            structured_data = self.parse(raw_data)
            records_parsed = len(structured_data)

            self.log.info("parse_complete", count=records_parsed)

            # 3. Validate
            valid_records, invalid_records = self.validate(structured_data)
            records_validated = len(valid_records)
            records_failed = len(invalid_records)

            errors.extend([
                {
                    "stage": "validation",
                    "record": inv["record"],
                    "errors": inv["errors"]
                }
                for inv in invalid_records
            ])

            self.log.info(
                "validation_complete",
                valid=records_validated,
                invalid=records_failed
            )

            # 4. Deduplicate
            unique_records, duplicate_count = self.deduplicate(valid_records)

            # 5. Store
            records_stored = self.store(unique_records)

            # 6. Emit events
            self.emit_events(unique_records)

            completed_at = datetime.now()
            status = "success" if records_failed == 0 else "partial"

            result = CollectionResult(
                collector_type=self.config.collector_type,
                run_id=run_id,
                started_at=started_at,
                completed_at=completed_at,
                status=status,
                records_fetched=records_fetched,
                records_parsed=records_parsed,
                records_validated=records_validated,
                records_stored=records_stored,
                records_duplicate=duplicate_count,
                records_failed=records_failed,
                errors=errors,
                metadata={
                    "since": since.isoformat(),
                    "duration_seconds": (completed_at - started_at).total_seconds()
                }
            )

            self.log.info(
                "collection_complete",
                run_id=run_id,
                status=status,
                duration_seconds=result.metadata["duration_seconds"],
                records_stored=records_stored
            )

            return result

        except Exception as e:
            completed_at = datetime.now()

            self.log.error(
                "collection_failed",
                run_id=run_id,
                error=str(e),
                error_type=type(e).__name__
            )

            return CollectionResult(
                collector_type=self.config.collector_type,
                run_id=run_id,
                started_at=started_at,
                completed_at=completed_at,
                status="failed",
                records_fetched=0,
                records_parsed=0,
                records_validated=0,
                records_stored=0,
                records_duplicate=0,
                records_failed=0,
                errors=[{
                    "stage": "pipeline",
                    "error": str(e),
                    "error_type": type(e).__name__
                }],
                metadata={}
            )
```

## Data Type Specifications

### 1. Insider Trades (Form 4)

**Data Source**: SEC EDGAR API
**Freshness**: Near real-time (15-30 minute delay)
**Schema**: `customs/schemas/form_4.json`

```python
# watchtower/collectors/form_4.py
from watchtower.collectors.base import BaseCollector, CollectorType
from datetime import datetime
from typing import Any, Dict, List
import requests


class Form4Collector(BaseCollector):
    """
    Collects insider trading data from SEC Form 4 filings.

    Form 4 reports:
    - Insider purchases/sales by corporate officers, directors, 10% owners
    - Transaction date, price, quantity
    - Post-transaction ownership

    Data characteristics:
    - Legally required within 2 business days of transaction
    - High signal value for certain insiders
    - Needs enrichment (e.g., insider track record)
    """

    def __init__(self, config):
        super().__init__(config)
        self.sec_api_base = config.source_config.get(
            "sec_api_base",
            "https://www.sec.gov/cgi-bin/browse-edgar"
        )
        self.user_agent = config.source_config.get(
            "user_agent",
            "Watchtower/1.0"
        )

    def fetch(self, since: datetime) -> List[Any]:
        """Fetch Form 4 filings from SEC EDGAR."""
        # Implementation would use SEC API or RSS feed
        params = {
            "action": "getcompany",
            "type": "4",
            "dateb": since.strftime("%Y%m%d"),
            "owner": "include",
            "output": "atom",
            "count": 100
        }

        response = requests.get(
            self.sec_api_base,
            params=params,
            headers={"User-Agent": self.user_agent}
        )
        response.raise_for_status()

        # Parse RSS/Atom feed
        # Return list of filing URLs
        return self._parse_feed(response.content)

    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """Parse Form 4 XML into structured format."""
        structured = []

        for filing_url in raw_data:
            # Fetch and parse XML
            form4_data = self._fetch_and_parse_form4(filing_url)

            # Extract transactions
            for transaction in form4_data.get("transactions", []):
                structured.append({
                    "id": f"form4-{form4_data['accession_number']}-{transaction['id']}",
                    "data_type": "form_4",
                    "filing_date": form4_data["filing_date"],
                    "accession_number": form4_data["accession_number"],

                    # Issuer (company)
                    "issuer": {
                        "cik": form4_data["issuer_cik"],
                        "name": form4_data["issuer_name"],
                        "ticker": form4_data.get("ticker"),
                    },

                    # Reporting owner (insider)
                    "reporting_owner": {
                        "cik": form4_data["owner_cik"],
                        "name": form4_data["owner_name"],
                        "relationship": form4_data["owner_relationship"],  # officer, director, 10% owner
                        "title": form4_data.get("owner_title"),
                    },

                    # Transaction
                    "transaction": {
                        "date": transaction["date"],
                        "type": transaction["type"],  # P=purchase, S=sale, A=award, etc.
                        "security_title": transaction["security_title"],
                        "shares": transaction["shares"],
                        "price_per_share": transaction.get("price"),
                        "acquired_disposed": transaction["acquired_disposed"],  # A or D
                        "transaction_code": transaction["code"],
                    },

                    # Post-transaction ownership
                    "post_transaction": {
                        "shares_owned": transaction["shares_owned_following"],
                        "direct_indirect": transaction["ownership_nature"],
                    },

                    # Metadata
                    "collected_at": datetime.now().isoformat(),
                    "source_url": filing_url,
                })

        return structured

    def get_schema_name(self) -> str:
        return "form_4"

    def _parse_feed(self, content: bytes) -> List[str]:
        """Parse SEC RSS/Atom feed."""
        # Implementation details...
        pass

    def _fetch_and_parse_form4(self, filing_url: str) -> Dict[str, Any]:
        """Fetch and parse individual Form 4 XML."""
        # Implementation details...
        pass
```

**Collection Schedule**: Every 15 minutes during market hours

**Key Signals**:
- Open market purchases by CEO/CFO
- Large transactions by directors
- Clusters of insider buying
- Insider selling near 52-week highs

### 2. Institutional Holdings (13F)

**Data Source**: SEC EDGAR 13F filings
**Freshness**: Quarterly (45-day lag)
**Schema**: `customs/schemas/form_13f.json`

```python
# watchtower/collectors/form_13f.py
from watchtower.collectors.base import BaseCollector


class Form13FCollector(BaseCollector):
    """
    Collects institutional holdings from SEC Form 13F filings.

    Form 13F reports:
    - Quarterly holdings of institutions managing >$100M
    - Long positions only (no shorts)
    - Holdings as of quarter end, filed within 45 days

    Data characteristics:
    - Significant lag (45 days)
    - High-conviction holdings of smart money
    - Useful for identifying consensus positions
    - Changes (additions/reductions) more valuable than absolute holdings
    """

    def fetch(self, since: datetime) -> List[Any]:
        """Fetch 13F filings from SEC."""
        # Query SEC for 13F filings
        # Focus on notable institutional investors (e.g., Berkshire, Soros, etc.)
        pass

    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """Parse 13F XML/text into structured format."""
        structured = []

        for filing in raw_data:
            filing_data = self._parse_13f(filing)

            for holding in filing_data["holdings"]:
                structured.append({
                    "id": f"13f-{filing_data['accession']}-{holding['cusip']}",
                    "data_type": "form_13f",
                    "filing_date": filing_data["filing_date"],
                    "period_of_report": filing_data["period_end"],
                    "accession_number": filing_data["accession"],

                    # Filer (institutional investor)
                    "filer": {
                        "cik": filing_data["filer_cik"],
                        "name": filing_data["filer_name"],
                    },

                    # Holding
                    "holding": {
                        "cusip": holding["cusip"],
                        "name": holding["name"],
                        "title_of_class": holding["title"],
                        "shares": holding["shares"],
                        "value": holding["value"],  # In thousands
                        "put_call": holding.get("put_call"),  # Optional
                        "investment_discretion": holding["discretion"],  # SOLE, SHARED, NONE
                        "voting_authority": holding["voting"],
                    },

                    # Metadata
                    "collected_at": datetime.now().isoformat(),
                    "source_url": filing["url"],
                })

        return structured

    def get_schema_name(self) -> str:
        return "form_13f"
```

**Collection Schedule**: Daily (to catch new filings), with priority processing for top institutions

**Key Signals**:
- New positions by notable investors
- Significant increases in existing positions
- Complete exits from positions
- Clustering of institutions in same stock

### 3. Earnings Data

**Data Source**: Multiple (Alpha Vantage, Financial Modeling Prep, Yahoo Finance)
**Freshness**: Real-time to daily
**Schema**: `customs/schemas/earnings.json`

```python
# watchtower/collectors/earnings.py
from watchtower.collectors.base import BaseCollector


class EarningsCollector(BaseCollector):
    """
    Collects earnings announcements and estimates.

    Data collected:
    - Earnings release dates (forward-looking calendar)
    - Actual earnings results (EPS, revenue)
    - Analyst estimates (consensus)
    - Earnings surprises
    - Guidance changes

    Data characteristics:
    - Calendar data available weeks in advance
    - Actual results within minutes of release
    - High impact on stock prices
    """

    def fetch(self, since: datetime) -> List[Any]:
        """Fetch earnings calendar and results."""
        # Fetch from multiple sources for redundancy
        # 1. Earnings calendar (upcoming)
        # 2. Recent earnings results
        pass

    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """Parse earnings data into structured format."""
        structured = []

        for item in raw_data:
            structured.append({
                "id": f"earnings-{item['ticker']}-{item['fiscal_quarter']}",
                "data_type": "earnings",

                # Company
                "company": {
                    "ticker": item["ticker"],
                    "name": item["company_name"],
                },

                # Reporting period
                "fiscal_quarter": item["fiscal_quarter"],  # e.g., "2024-Q4"
                "fiscal_year": item["fiscal_year"],

                # Timing
                "report_date": item["report_date"],
                "report_time": item.get("report_time"),  # BMO, AMC, or specific time

                # Estimates (if available)
                "estimates": {
                    "eps_consensus": item.get("eps_consensus"),
                    "eps_high": item.get("eps_high"),
                    "eps_low": item.get("eps_low"),
                    "revenue_consensus": item.get("revenue_consensus"),
                },

                # Actuals (if reported)
                "actuals": {
                    "eps_reported": item.get("eps_reported"),
                    "revenue_reported": item.get("revenue_reported"),
                    "eps_surprise_pct": item.get("eps_surprise_pct"),
                    "revenue_surprise_pct": item.get("revenue_surprise_pct"),
                },

                # Guidance (if provided)
                "guidance": {
                    "next_quarter_eps": item.get("guidance_eps"),
                    "next_quarter_revenue": item.get("guidance_revenue"),
                    "full_year_eps": item.get("fy_guidance_eps"),
                    "full_year_revenue": item.get("fy_guidance_revenue"),
                },

                # Metadata
                "collected_at": datetime.now().isoformat(),
                "data_sources": item["sources"],
            })

        return structured

    def get_schema_name(self) -> str:
        return "earnings"
```

**Collection Schedule**:
- Calendar: Daily
- Results: Real-time during earnings season (before market open, after close)

**Key Signals**:
- Large earnings surprises
- Guidance raises/cuts
- Earnings announcements for positions held
- Pre-earnings position sizing

### 4. News Data

**Data Source**: News APIs (NewsAPI, Benzinga, Reuters)
**Freshness**: Real-time
**Schema**: `customs/schemas/news.json`

```python
# watchtower/collectors/news.py
from watchtower.collectors.base import BaseCollector


class NewsCollector(BaseCollector):
    """
    Collects financial news articles.

    Data collected:
    - News headlines and content
    - Source and author
    - Mentioned tickers/companies
    - Article sentiment (if provided by API)

    Data characteristics:
    - High volume
    - Noisy signal
    - Requires filtering for relevance
    - Sentiment extraction needed
    """

    def __init__(self, config):
        super().__init__(config)
        self.api_key = config.source_config["api_key"]
        self.sources = config.source_config.get("sources", ["reuters", "bloomberg"])
        self.keywords = config.source_config.get("keywords", [])

    def fetch(self, since: datetime) -> List[Any]:
        """Fetch news articles."""
        # Fetch from news API
        # Filter by sources, keywords, tickers
        pass

    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """Parse news articles into structured format."""
        structured = []

        for article in raw_data:
            structured.append({
                "id": f"news-{article['id']}",
                "data_type": "news",

                # Article
                "title": article["title"],
                "content": article.get("content"),
                "summary": article.get("summary"),
                "url": article["url"],
                "published_at": article["published_at"],

                # Source
                "source": {
                    "name": article["source"],
                    "author": article.get("author"),
                },

                # Symbols mentioned
                "symbols": article.get("symbols", []),  # Tickers mentioned

                # Sentiment (if available from API)
                "sentiment": {
                    "score": article.get("sentiment_score"),  # -1 to 1
                    "label": article.get("sentiment_label"),  # positive, negative, neutral
                },

                # Categories
                "categories": article.get("categories", []),  # earnings, M&A, regulatory, etc.

                # Metadata
                "collected_at": datetime.now().isoformat(),
            })

        return structured

    def get_schema_name(self) -> str:
        return "news"
```

**Collection Schedule**: Every 5 minutes

**Key Signals**:
- Breaking news on held positions
- Sentiment shifts
- Event detection (M&A, FDA approvals, etc.)

### 5. Social Sentiment

**Data Source**: Twitter/X API, Reddit API, StockTwits
**Freshness**: Real-time
**Schema**: `customs/schemas/social_sentiment.json`

```python
# watchtower/collectors/social_sentiment.py
from watchtower.collectors.base import BaseCollector


class SocialSentimentCollector(BaseCollector):
    """
    Collects social media sentiment for stocks.

    Data collected:
    - Social media posts mentioning tickers
    - Aggregated sentiment metrics
    - Volume/buzz metrics
    - Influential accounts' opinions

    Data characteristics:
    - Very high volume
    - Extremely noisy
    - Requires aggressive filtering
    - Short-term predictive value
    """

    def fetch(self, since: datetime) -> List[Any]:
        """Fetch social sentiment data."""
        # Option 1: Use aggregated sentiment API (e.g., StockTwits sentiment)
        # Option 2: Fetch individual posts and aggregate ourselves
        pass

    def parse(self, raw_data: List[Any]) -> List[Dict[str, Any]]:
        """Parse social data into aggregated sentiment."""
        structured = []

        # Aggregate by ticker and time window
        for ticker_sentiment in raw_data:
            structured.append({
                "id": f"sentiment-{ticker_sentiment['ticker']}-{ticker_sentiment['timestamp']}",
                "data_type": "social_sentiment",

                # Symbol
                "ticker": ticker_sentiment["ticker"],

                # Time window
                "timestamp": ticker_sentiment["timestamp"],
                "window_minutes": ticker_sentiment["window"],  # e.g., 15, 60

                # Volume metrics
                "message_count": ticker_sentiment["message_count"],
                "unique_authors": ticker_sentiment["unique_authors"],
                "reach": ticker_sentiment.get("reach"),  # Based on follower counts

                # Sentiment metrics
                "sentiment": {
                    "score_mean": ticker_sentiment["sentiment_mean"],  # -1 to 1
                    "score_stddev": ticker_sentiment["sentiment_stddev"],
                    "positive_pct": ticker_sentiment["positive_pct"],
                    "negative_pct": ticker_sentiment["negative_pct"],
                    "neutral_pct": ticker_sentiment["neutral_pct"],
                },

                # Trend
                "trend": {
                    "vs_prev_window": ticker_sentiment.get("trend_prev"),
                    "vs_24h_avg": ticker_sentiment.get("trend_24h"),
                },

                # Top messages
                "notable_messages": ticker_sentiment.get("top_messages", []),

                # Metadata
                "collected_at": datetime.now().isoformat(),
                "sources": ticker_sentiment["sources"],  # twitter, reddit, stocktwits
            })

        return structured

    def get_schema_name(self) -> str:
        return "social_sentiment"
```

**Collection Schedule**: Every 15 minutes (with aggregation)

**Key Signals**:
- Unusual spikes in buzz
- Sentiment reversals
- Influencer mentions

## Orchestration Layer

### Collector Registry

```python
# watchtower/orchestration/registry.py
from typing import Dict, Type
from watchtower.collectors.base import BaseCollector, CollectorType
from watchtower.collectors.congressional import CongressionalCollector
from watchtower.collectors.form_4 import Form4Collector
from watchtower.collectors.form_13f import Form13FCollector
from watchtower.collectors.earnings import EarningsCollector
from watchtower.collectors.news import NewsCollector
from watchtower.collectors.social_sentiment import SocialSentimentCollector


class CollectorRegistry:
    """Registry of all available collectors."""

    _collectors: Dict[CollectorType, Type[BaseCollector]] = {
        CollectorType.CONGRESSIONAL: CongressionalCollector,
        CollectorType.FORM_4: Form4Collector,
        CollectorType.FORM_13F: Form13FCollector,
        CollectorType.EARNINGS: EarningsCollector,
        CollectorType.NEWS: NewsCollector,
        CollectorType.SENTIMENT: SocialSentimentCollector,
    }

    @classmethod
    def get_collector_class(cls, collector_type: CollectorType) -> Type[BaseCollector]:
        """Get collector class by type."""
        if collector_type not in cls._collectors:
            raise ValueError(f"Unknown collector type: {collector_type}")
        return cls._collectors[collector_type]

    @classmethod
    def register(cls, collector_type: CollectorType, collector_class: Type[BaseCollector]):
        """Register a new collector type."""
        cls._collectors[collector_type] = collector_class
```

### Scheduler

```python
# watchtower/orchestration/scheduler.py
from apscheduler.schedulers.background import BackgroundScheduler
from watchtower.orchestration.registry import CollectorRegistry
from watchtower.collectors.base import CollectionConfig
import structlog

log = structlog.get_logger()


class CollectorScheduler:
    """Schedules and runs collectors based on configuration."""

    def __init__(self, configs: list[CollectionConfig]):
        self.configs = {c.collector_type: c for c in configs}
        self.scheduler = BackgroundScheduler()
        self.registry = CollectorRegistry()

    def start(self):
        """Start all enabled collectors."""
        for collector_type, config in self.configs.items():
            if not config.enabled:
                log.info("collector_disabled", type=collector_type.value)
                continue

            # Get collector class
            collector_class = self.registry.get_collector_class(collector_type)

            # Create collector instance
            collector = collector_class(config)

            # Schedule collector
            self.scheduler.add_job(
                collector.run,
                trigger="cron",
                **self._parse_cron(config.schedule),
                id=f"collector-{collector_type.value}",
                replace_existing=True,
                max_instances=1,  # Prevent overlapping runs
            )

            log.info(
                "collector_scheduled",
                type=collector_type.value,
                schedule=config.schedule
            )

        self.scheduler.start()
        log.info("scheduler_started")

    def stop(self):
        """Stop scheduler."""
        self.scheduler.shutdown()
        log.info("scheduler_stopped")

    def _parse_cron(self, cron_expression: str) -> dict:
        """Parse cron expression into APScheduler kwargs."""
        # Implementation...
        pass
```

### Configuration

```yaml
# watchtower/config/collectors.yaml
collectors:
  congressional:
    enabled: true
    schedule: "*/30 * * * *"  # Every 30 minutes
    timeout: 300
    retry_attempts: 3
    retry_backoff: 2.0
    batch_size: 100
    lookback_days: 7
    requests_per_minute: 10
    requests_per_hour: 100
    source_config:
      api_url: "https://clerk.house.gov/public_disc/financial-pdfs"

  form_4:
    enabled: true
    schedule: "*/15 * * * *"  # Every 15 minutes
    timeout: 600
    retry_attempts: 3
    retry_backoff: 2.0
    batch_size: 200
    lookback_days: 2
    requests_per_minute: 5  # SEC rate limit
    requests_per_hour: 60
    source_config:
      sec_api_base: "https://www.sec.gov/cgi-bin/browse-edgar"
      user_agent: "Watchtower/1.0 (contact@example.com)"

  form_13f:
    enabled: true
    schedule: "0 2 * * *"  # Daily at 2 AM
    timeout: 1800
    retry_attempts: 3
    retry_backoff: 2.0
    batch_size: 50
    lookback_days: 90
    requests_per_minute: 5
    requests_per_hour: 60
    source_config:
      sec_api_base: "https://www.sec.gov/cgi-bin/browse-edgar"
      notable_filers:  # Prioritize these institutions
        - "0001067983"  # Berkshire Hathaway
        - "0001029160"  # Soros Fund Management

  earnings:
    enabled: true
    schedule: "*/5 * * * *"  # Every 5 minutes during market hours
    timeout: 120
    retry_attempts: 3
    retry_backoff: 1.5
    batch_size: 500
    lookback_days: 1
    requests_per_minute: 20
    requests_per_hour: 300
    source_config:
      primary_source: "alpha_vantage"
      api_key: "${ALPHA_VANTAGE_API_KEY}"
      fallback_sources:
        - "yahoo_finance"
        - "financial_modeling_prep"

  news:
    enabled: true
    schedule: "*/5 * * * *"  # Every 5 minutes
    timeout: 60
    retry_attempts: 2
    retry_backoff: 1.5
    batch_size: 1000
    lookback_days: 1
    requests_per_minute: 30
    requests_per_hour: 500
    source_config:
      api_provider: "newsapi"
      api_key: "${NEWS_API_KEY}"
      sources:
        - "reuters"
        - "bloomberg"
        - "cnbc"
      keywords:
        - "earnings"
        - "FDA approval"
        - "merger"
        - "acquisition"

  social_sentiment:
    enabled: false  # Disabled by default (experimental)
    schedule: "*/15 * * * *"  # Every 15 minutes
    timeout: 120
    retry_attempts: 2
    retry_backoff: 1.5
    batch_size: 2000
    lookback_days: 1
    requests_per_minute: 10
    requests_per_hour: 100
    source_config:
      sources:
        - "stocktwits"
        - "twitter"
      min_message_count: 50  # Only collect if sufficient volume
      watchlist_only: true   # Only track tickers in watchlist
```

## Storage Layer

### Database Schema

```sql
-- watchtower/schema/tables.sql

-- Main data table (flexible JSONB for different data types)
CREATE TABLE collected_data (
    id TEXT PRIMARY KEY,
    data_type TEXT NOT NULL,  -- congressional, form_4, etc.
    collected_at TIMESTAMPTZ NOT NULL,
    data JSONB NOT NULL,

    -- Extraction indexes for common queries
    ticker TEXT GENERATED ALWAYS AS (
        CASE
            WHEN data->>'ticker' IS NOT NULL THEN data->>'ticker'
            WHEN data->'issuer'->>'ticker' IS NOT NULL THEN data->'issuer'->>'ticker'
            ELSE NULL
        END
    ) STORED,

    event_date DATE GENERATED ALWAYS AS (
        COALESCE(
            (data->>'filing_date')::DATE,
            (data->>'transaction_date')::DATE,
            (data->>'report_date')::DATE,
            (data->>'published_at')::DATE
        )
    ) STORED,

    -- Metadata
    source_url TEXT,
    schema_version TEXT,

    CONSTRAINT valid_data_type CHECK (
        data_type IN ('congressional', 'form_4', 'form_13f', 'earnings', 'news', 'social_sentiment')
    )
);

-- Indexes
CREATE INDEX idx_data_type_date ON collected_data(data_type, event_date DESC);
CREATE INDEX idx_ticker_date ON collected_data(ticker, event_date DESC) WHERE ticker IS NOT NULL;
CREATE INDEX idx_collected_at ON collected_data(collected_at DESC);
CREATE INDEX idx_data_gin ON collected_data USING GIN (data);

-- Collection runs (for monitoring)
CREATE TABLE collection_runs (
    run_id TEXT PRIMARY KEY,
    collector_type TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL,  -- success, partial, failed
    records_fetched INT,
    records_stored INT,
    records_failed INT,
    errors JSONB,
    metadata JSONB
);

CREATE INDEX idx_runs_collector_time ON collection_runs(collector_type, started_at DESC);
```

## Error Handling and Resilience

### Retry Strategy

```python
# watchtower/resilience/retry.py
import time
from typing import Callable, Any
import structlog

log = structlog.get_logger()


def with_retry(
    func: Callable,
    max_attempts: int = 3,
    backoff: float = 2.0,
    exceptions: tuple = (Exception,)
) -> Any:
    """Execute function with exponential backoff retry."""
    for attempt in range(1, max_attempts + 1):
        try:
            return func()
        except exceptions as e:
            if attempt == max_attempts:
                log.error(
                    "retry_exhausted",
                    function=func.__name__,
                    attempts=max_attempts,
                    error=str(e)
                )
                raise

            wait_time = backoff ** attempt
            log.warning(
                "retry_attempt",
                function=func.__name__,
                attempt=attempt,
                max_attempts=max_attempts,
                wait_seconds=wait_time,
                error=str(e)
            )
            time.sleep(wait_time)
```

### Rate Limiting

```python
# watchtower/resilience/rate_limiter.py
import time
from collections import deque
from datetime import datetime, timedelta


class RateLimiter:
    """Token bucket rate limiter."""

    def __init__(self, requests_per_minute: int, requests_per_hour: int):
        self.rpm = requests_per_minute
        self.rph = requests_per_hour
        self.minute_window = deque()
        self.hour_window = deque()

    def acquire(self):
        """Wait if necessary to respect rate limits."""
        now = datetime.now()

        # Clean old entries
        cutoff_minute = now - timedelta(minutes=1)
        cutoff_hour = now - timedelta(hours=1)

        while self.minute_window and self.minute_window[0] < cutoff_minute:
            self.minute_window.popleft()

        while self.hour_window and self.hour_window[0] < cutoff_hour:
            self.hour_window.popleft()

        # Check limits
        if len(self.minute_window) >= self.rpm:
            # Wait until oldest request in minute window expires
            wait_until = self.minute_window[0] + timedelta(minutes=1)
            sleep_time = (wait_until - now).total_seconds()
            if sleep_time > 0:
                time.sleep(sleep_time)

        if len(self.hour_window) >= self.rph:
            # Wait until oldest request in hour window expires
            wait_until = self.hour_window[0] + timedelta(hours=1)
            sleep_time = (wait_until - now).total_seconds()
            if sleep_time > 0:
                time.sleep(sleep_time)

        # Record this request
        now = datetime.now()
        self.minute_window.append(now)
        self.hour_window.append(now)
```

## Monitoring and Observability

### Metrics

```python
# watchtower/monitoring/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Collection metrics
collections_total = Counter(
    'watchtower_collections_total',
    'Total collection runs',
    ['collector_type', 'status']
)

records_collected = Counter(
    'watchtower_records_collected_total',
    'Total records collected',
    ['collector_type', 'stage']
)

collection_duration = Histogram(
    'watchtower_collection_duration_seconds',
    'Collection duration',
    ['collector_type']
)

collection_errors = Counter(
    'watchtower_collection_errors_total',
    'Collection errors',
    ['collector_type', 'error_type']
)

# Data freshness
data_freshness = Gauge(
    'watchtower_data_freshness_seconds',
    'Time since last successful collection',
    ['collector_type']
)
```

### Health Checks

```python
# watchtower/monitoring/health.py
from datetime import datetime, timedelta
from watchtower.storage.reader import DataReader


class HealthChecker:
    """Check health of data collection."""

    def check_all(self) -> dict:
        """Run all health checks."""
        return {
            "collectors": self.check_collectors(),
            "data_freshness": self.check_data_freshness(),
            "storage": self.check_storage(),
        }

    def check_data_freshness(self) -> dict:
        """Check if data is fresh for all collectors."""
        reader = DataReader()
        freshness = {}

        for collector_type in CollectorType:
            latest = reader.get_latest_record(collector_type)
            if latest:
                age = datetime.now() - latest["collected_at"]
                freshness[collector_type.value] = {
                    "status": "healthy" if age < timedelta(hours=1) else "stale",
                    "age_seconds": age.total_seconds(),
                    "latest_record_time": latest["collected_at"].isoformat()
                }
            else:
                freshness[collector_type.value] = {
                    "status": "no_data",
                    "age_seconds": None,
                    "latest_record_time": None
                }

        return freshness
```

## Integration with Bartertown

### Data Access API

```python
# watchtower/api/data_access.py
from datetime import datetime, timedelta
from typing import List, Optional, Dict, Any
from watchtower.storage.reader import DataReader


class WatchtowerAPI:
    """
    API for Bartertown to access Watchtower data.

    Design principles:
    - Read-only access
    - Efficient queries
    - Type-safe returns
    - Caching-friendly
    """

    def __init__(self):
        self.reader = DataReader()

    def get_congressional_trades(
        self,
        since: Optional[datetime] = None,
        ticker: Optional[str] = None,
        chamber: Optional[str] = None,
        limit: int = 100
    ) -> List[Dict[str, Any]]:
        """Get congressional trades with filters."""
        return self.reader.query(
            data_type="congressional",
            since=since or datetime.now() - timedelta(days=30),
            filters={"ticker": ticker, "chamber": chamber},
            limit=limit
        )

    def get_insider_trades(
        self,
        ticker: str,
        since: Optional[datetime] = None,
        transaction_type: Optional[str] = None
    ) -> List[Dict[str, Any]]:
        """Get insider trades for a ticker."""
        return self.reader.query(
            data_type="form_4",
            since=since or datetime.now() - timedelta(days=90),
            filters={"ticker": ticker, "transaction_type": transaction_type},
            limit=1000
        )

    def get_earnings_calendar(
        self,
        start_date: datetime,
        end_date: datetime,
        tickers: Optional[List[str]] = None
    ) -> List[Dict[str, Any]]:
        """Get earnings calendar for date range."""
        return self.reader.query(
            data_type="earnings",
            date_range=(start_date, end_date),
            filters={"tickers": tickers} if tickers else {},
            limit=10000
        )

    def get_institutional_holdings(
        self,
        ticker: str,
        top_n_filers: int = 20
    ) -> List[Dict[str, Any]]:
        """Get institutional holdings for a ticker from top filers."""
        return self.reader.query(
            data_type="form_13f",
            filters={"ticker": ticker},
            sort_by="filer_rank",
            limit=top_n_filers
        )
```

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
1. Implement BaseCollector interface
2. Build validation, deduplication, storage layers
3. Create orchestration/scheduling system
4. Set up monitoring and logging

### Phase 2: Core Collectors (Weeks 3-4)
1. Congressional collector (migrate existing)
2. Form 4 collector
3. Earnings collector

### Phase 3: Extended Collectors (Weeks 5-6)
1. Form 13F collector
2. News collector
3. Social sentiment collector (experimental)

### Phase 4: Integration (Week 7)
1. Build Watchtower API for Bartertown
2. Integration testing
3. Performance optimization

### Phase 5: Production (Week 8)
1. Deploy to production
2. Monitor data quality
3. Iterate based on usage

## Success Metrics

### Data Quality
- 99%+ successful collection rate
- <1% invalid records (schema violations)
- <0.1% duplicate records
- <1 hour data freshness (for real-time collectors)

### System Reliability
- 99.9% uptime
- <5 minute recovery from collector failures
- <10% retry rate

### Performance
- <1 minute collection time for fast collectors
- <10 minute collection time for batch collectors
- <100ms query latency for data access API

## Appendix A: Customs Schema Examples

### Form 4 Schema (JSON Schema)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Form 4 Insider Transaction",
  "type": "object",
  "required": ["id", "data_type", "filing_date", "issuer", "reporting_owner", "transaction"],
  "properties": {
    "id": {"type": "string"},
    "data_type": {"const": "form_4"},
    "filing_date": {"type": "string", "format": "date"},
    "accession_number": {"type": "string"},
    "issuer": {
      "type": "object",
      "required": ["cik", "name"],
      "properties": {
        "cik": {"type": "string"},
        "name": {"type": "string"},
        "ticker": {"type": "string"}
      }
    },
    "reporting_owner": {
      "type": "object",
      "required": ["cik", "name", "relationship"],
      "properties": {
        "cik": {"type": "string"},
        "name": {"type": "string"},
        "relationship": {"type": "string"},
        "title": {"type": "string"}
      }
    },
    "transaction": {
      "type": "object",
      "required": ["date", "type", "shares", "acquired_disposed"],
      "properties": {
        "date": {"type": "string", "format": "date"},
        "type": {"type": "string"},
        "security_title": {"type": "string"},
        "shares": {"type": "number"},
        "price_per_share": {"type": "number"},
        "acquired_disposed": {"enum": ["A", "D"]},
        "transaction_code": {"type": "string"}
      }
    }
  }
}
```

## Appendix B: Adding a New Collector

### Quick Start Guide

1. **Create collector class**:
```python
# watchtower/collectors/my_collector.py
from watchtower.collectors.base import BaseCollector

class MyCollector(BaseCollector):
    def fetch(self, since):
        # Fetch raw data
        pass

    def parse(self, raw_data):
        # Parse to structured format
        pass

    def get_schema_name(self):
        return "my_data_type"
```

2. **Define Customs schema**:
```json
// customs/schemas/my_data_type.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "My Data Type",
  ...
}
```

3. **Register collector**:
```python
# In watchtower/orchestration/registry.py
CollectorRegistry.register(
    CollectorType.MY_TYPE,
    MyCollector
)
```

4. **Add configuration**:
```yaml
# watchtower/config/collectors.yaml
collectors:
  my_type:
    enabled: true
    schedule: "*/30 * * * *"
    ...
```

5. **Test**:
```python
# tests/collectors/test_my_collector.py
def test_my_collector():
    collector = MyCollector(config)
    result = collector.run()
    assert result.status == "success"
```

---

**Document Status**: Planning Phase
**Owner**: bartertown/polecats/capable
**Related Beads**: bt-xy3 (this task), bt-8mn (congressional ingestion), bt-4q8 (synthesis)
**Next Step**: Synthesis into unified architecture (bt-4q8)
