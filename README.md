# big5personalitymarketerion

function callChatGPTForAllRows() {
  const apiKey = 'example; // Replace with your actual API key
  const url = 'https://api.openai.com/v1/chat/completions';

  // Get the spreadsheet and sheets
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sumScoresSheet = spreadsheet.getSheetByName('Sumscores Respondents_PowerBI');
  if (!sumScoresSheet) {
    Logger.log('Sheet not found: Sumscores Respondents_PowerBI');
    return;
  }
  const formResponsesSheet = spreadsheet.getSheetByName('Raw Form Data Respondent');
  if (!formResponsesSheet) {
    Logger.log('Sheet not found: Form Responses');
    return;
  }
function onChange(e) {
  Logger.log("onChange Trigger Fired");
}

  Utilities.sleep(2000); // Wait if needed for calculations

  // Identify your data ranges
  const lastRow = sumScoresSheet.getLastRow();
  const data = sumScoresSheet.getRange(2, 7, lastRow - 1, 5).getValues(); // G:K (5 columns)
  const outputColumn = sumScoresSheet.getRange(2, 12, lastRow - 1, 1).getValues(); // Column L (output)

  // Columns BA (53) to BK (63) = 11 columns for open-ended questions
  const questionLabels = formResponsesSheet.getRange(1, 53, 1, 63 - 53 + 1).getValues()[0];

  function formatPercentage(value) {
    return (value * 100).toFixed(1) + "%"; // Converts 0.163 to "16.3%"
  }

  for (let i = 0; i < data.length; i++) {
    const row = data[i];
    const hasData = row.every(cell => cell && cell.toString().trim() !== "");

// Skip if personality data is incomplete
if (!hasData) {
  continue; // No log message, silently skips the row
}

// Skip if there's already output in column L
if (outputColumn[i][0] && outputColumn[i][0].trim() !== "") {
  continue; // No log message, silently skips the row
}


    // Get open-ended answers from Form Responses, columns BB:BL (54:64)
    const answersRow = formResponsesSheet.getRange(i + 2, 53, 1, 63 - 53 + 1).getValues()[0];

    // Build the text block for the open-ended questions and answers
    let openEndedBlock = '';
    for (let j = 0; j < questionLabels.length; j++) {
      const question = questionLabels[j];
      const answer = answersRow[j] ? answersRow[j].toString().trim() : '';
      if (question && answer) {
        openEndedBlock += `\n- ${question}: ${answer}`;
      }
    }

    // Format percentage scores
    const extroversion = formatPercentage(row[0]);
    const agreeableness = formatPercentage(row[1]);
    const conscientiousness = formatPercentage(row[2]);
    const emotionalStability = formatPercentage(row[3]);
    const openness = formatPercentage(row[4]);

    // Replace the old prompt construction with the new one inside the loop:
const prompt = `
I want to generate personalized career advice based on a Big Five personality profile (percentile scores) and additional information about education, work experience, and interests. Create a list of the top 10 jobs that best fit the respondent, ranked from highest to lowest suitability.

Use the following criteria:
- Education: Offer roles that align with the respondents education and prioritize interdisciplinary thinking if applicable.
- Work Experience: Consider current work experience and assess whether there are connections to the proposed roles. If not, suggest starting as a junior. If relevant experience exists, propose an appropriate seniority level.
- Interests: See where the respondent's stated interests align with their personality profile and prioritize roles that match sector preferences.
- Personality: Apply the Big Five percentile scores to suggest roles that would be a good fit for the respondent.

Big Five Scores:
- Extraversion: ${extroversion}
- Emotional Stability: ${emotionalStability}
- Agreeableness: ${agreeableness}
- Conscientiousness: ${conscientiousness}
- Openness to Experience: ${openness}

Additional information from the participant:
${openEndedBlock}

Output Structure:
First summarize the percentile scores for each dimension.
Then summarize what type of jobs would suit the respondent based on all the answers they provided.
Then, for each proposed job recommendation of the top 10, specify in the title whether the role should be a junior, medior, or senior position based on the respondents experience.
Then provide a  Job Description: Outline the responsibilities of the role and the key reasons why this is the best match for their personality and other answered questions.
Then do a deep dive on the alignment between each Personality Dimension and the suitability, provide examples. Give the strengths and weaknesses on each personality dimension for the job.
Always interject prior work experience, education level and interests out of the openEndedBlock where suitable.
- Make it so that it will be nice to read for the customer that will be given advice.
`;

    // Prepare ChatGPT request
    const requestBody = {
      model: 'gpt-4-turbo',
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        { role: 'user', content: prompt }
      ],
      max_tokens: 1000,
      temperature: 0.5
    };

    const options = {
      method: 'post',
      contentType: 'application/json',
      headers: {
        'Authorization': `Bearer ${apiKey}`
      },
      payload: JSON.stringify(requestBody),
      muteHttpExceptions: true
    };

    // Retry logic for API calls
    let retryCount = 0;
    const maxRetries = 3;

    while (retryCount < maxRetries) {
      try {
        const response = UrlFetchApp.fetch(url, options);
        const result = JSON.parse(response.getContentText());

        if (result.choices && result.choices.length > 0) {
          const chatGPTResponse = result.choices[0].message.content;
          sumScoresSheet.getRange(i + 2, 12).setValue(chatGPTResponse);
          Logger.log(`Row ${i + 2}: ChatGPT response written.`);
          break;
        } else {
          Logger.log(`Row ${i + 2}: No valid response received.`);
        }
      } catch (error) {
        retryCount++;
        Utilities.sleep(1000); // Wait before retrying
        if (retryCount === maxRetries) {
          Logger.log(`Row ${i + 2}: Failed after ${maxRetries} retries. Error: ${error.message}`);
          // Optionally, write the error to column M
          sumScoresSheet.getRange(i + 2, 13).setValue(`Error: ${error.message}`);
        }
      }
    }
  }
}
