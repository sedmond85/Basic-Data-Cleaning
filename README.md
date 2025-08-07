# Basic-Data-Cleaning
![Cleaning Bubbles](https://imagehost9.online-image-editor.com/oie_upload/images/723252a8Gs5X/TKXs04Ia51Nw.png)
##Turn messy data into clean, ready-to-use datasets
This project takes a messy dataset full of duplicates, weird formatting, missing data, and blank fields — and turns it into something polished and analysis-ready.
# 🧹 SQL Data Cleaning: Layoffs Dataset

## 🔧 What Was Done

- Cloned the raw table to work safely 🛡️
- Zapped duplicates using `ROW_NUMBER()` 💥
- Cleaned messy `company`, `industry`, and `country` fields 🧽
- Converted string `date` to real `DATE` ⏳
- Inferred missing industry data using self-joins 🔍
- Removed empty/useless records 🗑️
- Final result: a lean, clean dataset ready for insight 🚀

---

## 💻 Full SQL Script

```sql
-- 1. Create duplicate table
CREATE TABLE layoffs_staging LIKE layoffs;

INSERT INTO layoffs_staging
SELECT * FROM layoffs;

-- 2. Remove Duplicates
WITH duplicate_cte AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
    ) AS row_num
  FROM layoffs_staging
)
DELETE FROM layoffs_staging
WHERE id IN (
  SELECT id FROM duplicate_cte WHERE row_num > 1
);

-- 3. Standardize Data
UPDATE layoffs_staging SET company = TRIM(company);

UPDATE layoffs_staging
SET industry = 'Crypto'
WHERE industry LIKE '%crypto%';

UPDATE layoffs_staging
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE '%united states%';

UPDATE layoffs_staging
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoffs_staging
MODIFY COLUMN `date` DATE;

-- 4. Fill Null or Blank Industries
UPDATE layoffs_staging
SET industry = NULL
WHERE industry = '';

UPDATE layoffs_staging t1
JOIN layoffs_staging t2 ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;

-- 5. Remove Useless Records
DELETE FROM layoffs_staging
WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;

-- 6. Drop Helper Columns (if any)
ALTER TABLE layoffs_staging
DROP COLUMN row_num;

