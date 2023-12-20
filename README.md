# assiment
// main.go

package main

import (
	"encoding/json"
	"net/http"
	"sort"
	"sync"
	"time"
)

func main() {
	http.HandleFunc("/process-single", handleSingle)
	http.HandleFunc("/process-concurrent", handleConcurrent)
	http.ListenAndServe(":8000", nil)
}

func handleSingle(w http.ResponseWriter, r *http.Request) {
	processRequest(w, r, false)
}

func handleConcurrent(w http.ResponseWriter, r *http.Request) {
	processRequest(w, r, true)
}

func processRequest(w http.ResponseWriter, r *http.Request, concurrent bool) {
	var requestBody map[string][][]int
	err := json.NewDecoder(r.Body).Decode(&requestBody)
	if err != nil {
		http.Error(w, "Invalid input JSON", http.StatusBadRequest)
		return
	}

	toSort := requestBody["to_sort"]
	var sortedArrays [][]int

	startTime := time.Now()

	if concurrent {
		sortedArrays = sortConcurrently(toSort)
	} else {
		sortedArrays = sortSequentially(toSort)
	}

	timeTaken := time.Since(startTime).Nanoseconds()

	response := map[string]interface{}{
		"sorted_arrays": sortedArrays,
		"time_ns":       timeTaken,
	}

	responseJSON, err := json.Marshal(response)
	if err != nil {
		http.Error(w, "Internal server error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(responseJSON)
}

func sortSequentially(arrays [][]int) [][]int {
	var sortedArrays [][]int
	for _, arr := range arrays {
		sortedArr := make([]int, len(arr))
		copy(sortedArr, arr)
		sort.Ints(sortedArr)
		sortedArrays = append(sortedArrays, sortedArr)
	}
	return sortedArrays
}

func sortConcurrently(arrays [][]int) [][]int {
	var wg sync.WaitGroup
	var mu sync.Mutex
	sortedArrays := make([][]int, len(arrays))

	for i, arr := range arrays {
		wg.Add(1)
		go func(i int, arr []int) {
			defer wg.Done()

			sortedArr := make([]int, len(arr))
			copy(sortedArr, arr)
			sort.Ints(sortedArr)

			mu.Lock()
			sortedArrays[i] = sortedArr
			mu.Unlock()
		}(i, arr)
	}

	wg.Wait()
	return sortedArrays
}
